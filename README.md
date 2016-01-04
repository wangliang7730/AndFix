# AndFix

Android插件化是为了解决什么问题？

65536方法数爆棚。
减少apk体积。
不通过发版就能发布新功能
不通过发版就能修复线上bug和崩溃。


但我们经过实践发现，插件化更多的运用于修复线上bug和崩溃。这是一个很轻量的需求，却为此花费了大量的人力和财力去运行这样一套庞大的架构体系，是相当不划算的。为此，2015年github上出现了AndFix，这款Android轻量级线上bug修复工具。

AndFix的核心思想是，把app中的方法B替换为新的方法B ‘,为此，我们把新方法B’所在的代码进行重新打包，并和就的apk包进行差分比较，得到一个差分包，放到服务器提供旧版apk下载，那么apk在接收到差分包后，就会执行新的方法B’了，如下图所示：

这类似于iOS的JSPatch实现。只不过Objective-C是一门动态语言，天生就支持这样的特性。而在Android中，则需要修改Native底层了。

在Native底层，有一个dalvik_dispatcher方法负责最终执行哪个方法。就是在这里做一些手脚，把旧方法替换为新方法。

对于功能不是很多的App而言，AndFix是首选，可以快速修复线上bug而不用发新版，而实现成本也很低。对于规模不大的团队而言，相当划算。

这里不得不说到dexposed。dexposed和AndFix都是阿里推出的开源框架，用以解决Android热修复的两种实现，原理两者类似，都是在在Native底层的dalvik_dispatcher方法做文章。但是dexposed有一个硬伤，就是不支持art，这使得很多粉丝转而去投标AndFix阵营。dexposed的github地址为：

https://github.com/alibaba/dexposed

[![Download](https://api.bintray.com/packages/supern/maven/andfix/images/download.svg) ](https://bintray.com/supern/maven/andfix/_latestVersion)
[![Build Status](https://travis-ci.org/alibaba/AndFix.svg)](https://travis-ci.org/alibaba/AndFix)
[![Software License](https://rawgit.com/alibaba/AndFix/master/images/license.svg)](LICENSE)

[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/alibaba/AndFix?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

AndFix is a solution to fix the bugs online instead of redistributing Android App. It is distributed as [Android Library](https://sites.google.com/a/android.com/tools/tech-docs/new-build-system/aar-format).

Andfix is an acronym for "**And**roid hot-**fix**".

AndFix supports Android version from 2.3 to 6.0, both ARM and X86 architecture, both Dalvik and ART runtime.

The compressed file format of AndFix's patch is **.apatch**. It is dispatched from your own server to client to fix your App's bugs.

## Principle

The implementation principle of AndFix is method body's replacing,

![image](images/principle.png)

### Method replacing

AndFix judges the methods should be replaced by java custom annotation and replaces it by hooking it. AndFix has a native method `art_replaceMethod` in ART or `dalvik_replaceMethod` in X86 architecture. Implementations are different. For Dalvik, it will change the target method type to 'native' and link the method implementation to AndFix's own native, generic method called `dalvik_dispatcher`, this method then takes care of invoking callbacks that have been registered, as we say 'hooked'. For ART, we just change the `ArtMethod` properties of itself to replace it.

For more details, [here](https://github.com/alibaba/AndFix/tree/master/jni).

## Fix Process

![image](images/process.png)

## Integration

### How to get?

Directly add AndFix aar to your project as compile libraries.

For your maven dependency,

	```
	<dependency>
  		<groupId>com.alipay.euler</groupId>
  		<artifactId>andfix</artifactId>
  		<version>0.3.1</version>
  		<type>aar</type>
	</dependency>
	```
For your gradle dependency,

	```
	dependencies {
   		compile 'com.alipay.euler:andfix:0.3.1@aar'
	}
	```

### How to use?

1. Initialize PatchManager,

	```
	patchManager = new PatchManager(context);
	patchManager.init(appversion);//current version
	```

2. Load patch,

	```
	patchManager.loadPatch();
	```

	You should load patch as early as possible, generally, in the initialization phase of your application(such as `Application.onCreate()`).

3. Add patch,

	```
	patchManager.addPatch(path);//path of the patch file that was downloaded
	```
	When a new patch file has been downloaded, it will become effective immediately by `addPatch`.

## Developer Tool

AndFix provides a patch-making tool called **apkpatch**.

### How to get?

The `apkpatch` tool can be found [here](https://github.com/alibaba/AndFix/raw/master/tools/apkpatch-1.0.3.zip).

### How to use?

* Prepare two android packages, one is the online package, the other one is the package after you fix bugs by coding.

* Generate `.apatch` file by providing the two package,

```
usage: apkpatch -f <new> -t <old> -o <output> -k <keystore> -p <***> -a <alias> -e <***>
 -a,--alias <alias>     keystore entry alias.
 -e,--epassword <***>   keystore entry password.
 -f,--from <loc>        new Apk file path.
 -k,--keystore <loc>    keystore path.
 -n,--name <name>       patch name.
 -o,--out <dir>         output dir.
 -p,--kpassword <***>   keystore password.
 -t,--to <loc>          old Apk file path.
```

Now you get the application savior, the patch file. Then you need to dispatch it to your client in some way, push or pull.

Sometimes, your team members may fix each other's bugs, and generate not only one `.apatch`. For this situation, you can
merge `.apatch` files using this tool,

```
usage: apkpatch -m <apatch_path...> -o <output> -k <keystore> -p <***> -a <alias> -e <***>
 -a,--alias <alias>     keystore entry alias.
 -e,--epassword <***>   keystore entry password.
 -k,--keystore <loc>    keystore path.
 -m,--merge <loc...>    path of .apatch files.
 -n,--name <name>       patch name.
 -o,--out <dir>         output dir.
 -p,--kpassword <***>   keystore password.
```

## Running sample

1. Import samplesI/AndFixDemo to your IDE, append AndFixDemo dependencies with AndFix(library project or aar).
2. Build project, save the package as 1.apk, and then install on device/emulator.
3. Modify com.euler.test.A, references com.euler.test.Fix.
4. Build project, save the package as 2.apk.
5. Use apkpatch tool to make a patch.
6. Rename the patch file to out.apatch, and then copy it to sdcard.
7. Run 1.apk and view log.

## Notice

### ProGuard

If you enable ProGuard, you must save the mapping.txt, so your new version's build can use it with ["-applymapping"](http://proguard.sourceforge.net/manual/usage.html#applymapping).

And it is necessary to keep classes as follow,

* Native method

	com.alipay.euler.andfix.AndFix

* Annotation

	com.alipay.euler.andfix.annotation.MethodReplace

To ensure that these classes can be found after running an obfuscation and static analysis tool like ProGuard, add the configuration below to your ProGuard configuration file.


	```
	-keep class * extends java.lang.annotation.Annotation
	-keepclasseswithmembernames class * {
    	native <methods>;
	}
	```

### Self-Modifying Code

If you use it, such as *Bangcle*. To generate patch file, you'd better to use raw apk.

### Security

The following is important but out of AndFix's range.

-  verify the signature of patch file
-  verify the fingerprint of optimize file

## API Documentation

The libraries javadoc can be found [here](https://rawgit.com/alibaba/AndFix/master/docs/index.html).

## Contact

...

## License

[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.html)

Copyright (c) 2015, alipay.com
