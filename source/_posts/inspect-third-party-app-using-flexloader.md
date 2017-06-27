---
title: 使用 FLEX 调试任意第三方应用
date: 2017-06-04 17:30:29
tags: [逆向]
---

> 本文章的源码已经上传到 GitHub：[https://github.com/buginux/FLEXLoader](https://github.com/buginux/FLEXLoader)

最近[有一篇文章](http://ryanipete.com/blog/ios/swift/objective-c/uidebugginginformationoverlay/)很火，这篇文章的作者发现了苹果的一个私有方法，可以在应用中调出悬浮的调试窗口。调试工具很强大，可以在运行时查看视图层级，还能查看各个类的属性和实例变量，不得不说，这对开发者来说是一件利器。

但是，其实在这个私有方法被发现之前很久，就已经有一个开源的应用内调试工具可以提供类似甚至更强大的功能了，那就是 Flipboard 公司开源的 [FLEX](https://github.com/Flipboard/FLEX)。至于这个调试工具的功能有多逆天，大家可以自行上[它的 Github仓库](https://github.com/Flipboard/FLEX)进行查看，这里就不赘述了。

<!-- more -->

这个工具的功能是如此强大，如果只用来调试自己的应用，未免有点浪费。事实上，FLEX 的作者已经也在 Github 上明说了，它不仅能调试自己的应用，还能用来调试第三方的应用。不过他不告诉我们怎么做，这是留给读者的一个练习。

那今天，我们就来完成这个练习。

## 开始

由于这个工具本质上就是一个动态库，所以我们可以在应用启动的时候载入这个动态库，这样就能在任意应用中打开这个调试工具了。

但是，我们总不能为每一个想要使用 FLEX 进行调试的应用都写一个 Tweak 吧，这未免就太劳民伤财了。那如果不单独为每一个应用写 Tweak，我们又要如何做呢？

这时，我想到了 [RevealLoader](http://bbs.iosre.com/t/reveal-loader-reveal-app/187) 这个 Tweak。它是配合 Reveal 来查看第三方应用的 UI 的，也需要在程序启动时载入动态库，更厉害的是它可以让我们在系统的设置界面中选择特定的程序打开或者关闭这个功能，这正好就跟我们的需求一模一样了。

幸运的是，RevealLoader 的作者将这个 Tweak 开源了，因此我们有了一个借鉴的基础。

## AppList 和 PreferenceLoader

从 [RevealLoader](https://github.com/heardrwt/RevealLoader) 的源码中可以看到，它用到了 AppList 和 PreferenceLoader 两个依赖。

PreferenceLoader 是一个 MobileSubstrate 提供的工具，它可以让开发者在系统设置界面添加应用程序入口。而 AppList 是一个让开发者获取系统中已安装应用信息的库。

这两个工具可以跟完美配合，轻松在系统设置中实现可供选择的应用列表，效果如下：

![inspect-third-party-app-using-flexloader-pic-1](http://7xqonv.com1.z0.glb.clouddn.com/inspect-third-party-app-using-flexloader-pic-1.png)


要实现这个功能很简单，只需要在 iOS 设备的 `/Library/PreferenceLoader/Preferences` 下放入一个 `plist` 和三个图标文件（对应不同分辨率）。其中，`plist` 文件用来指定设置界面的展示内容，而图标文件则是用于在系统设置的入口处显示。

以下是参考 RevealLoader 并进行修改后的 `plist` 文件内容：

```
entry = {
  bundle = AppList;
  cell = PSLinkCell;
  icon = "/Library/PreferenceLoader/Preferences/XCode.png";
  isController = 1;
  label = FLEXLoader;
  ALSettingsPath = "/var/mobile/Library/Preferences/com.swiftyper.flexloader.plist";
  ALSettingsKeyPrefix = "FLEXLoaderEnabled-";
  ALChangeNotification = "com.swiftyper.flexloader.settingschanged";
  ALSettingsDefaultValue = 0;
  ALSectionDescriptors = (
  	{
  	  title = "User Applications";
  	  predicate = "(isSystemApplication = FALSE)";
  	  "cell-class-name" = "ALSwitchCell";
  	  "icon-size" = 29;
  	  "suppress-hidden-apps" = 1;
  	  
  	},
  	{
  	  title = "System Applications";
  	  predicate = "(isSystemApplication = TRUE)";
  	  "cell-class-name" = "ALSwitchCell";
  	  "icon-size" = 29;
  	  "suppress-hidden-apps" = 1;
  	  "footer-title" = "© buginux create for flexloader";
  	}
  );
};
```

比较重要的是两个字段是 `ALSettingsPath` 和 `ALSettingsKeyPrefix`，前者指定对应设置的保存路径，后者指定每个应用条目配置的前缀。这两个字段在之后的代码中都会使用到。

## 编译动态库

接下来就要从 FLEX 的源码中编译出 dylib 文件了，我们可以直接使用 XCode 进行编译，但是为了方便在 Tweak 的 makefile 中使用，我们需要一个可以编译出 dylib 的脚本，这个也不难，脚本的内容如下：

```bash
#!/usr/bin/env bash

echo "Cleaning up..."
rm -rf bin/ src/

echo "Updating submodules..."
git submodule update --init --recursive

echo "Copying sources..."
mkdir src/
find FLEX/Classes -type f \( -name "*.h" -o -name "*.m" \) -exec cp {} src/ \;

BIN_NAME="libFLEX.dylib"
IOS_VERSION_MIN=7.0

DEVELOPER_DIR="$(xcode-select -print-path)"
#DEVELOPER_DIR="/Applications/Xcode.app/Contents/Developer"
SDK_ROOT_OS=$DEVELOPER_DIR/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk
SDK_ROOT_SIMULATOR=$DEVELOPER_DIR/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk

ARCHS="armv7 arm64"
INPUT=$(find src -type f -name "*.m")

for ARCH in ${ARCHS}
do
	DIR=bin/${ARCH}
	mkdir -p ${DIR}
	echo "Building for ${ARCH}..."
	if [[ "${ARCH}" == "i386" || "${ARCH}" == "x86_64" ]];
	then
		SDK_ROOT=${SDK_ROOT_SIMULATOR}
		IOS_VERSION_MIN_FLAG=-mios-simulator-version-min
	else
		SDK_ROOT=${SDK_ROOT_OS}
		IOS_VERSION_MIN_FLAG=-mios-version-min
	fi
		FRAMEWORKS=${SDK_ROOT}/System/Library/Frameworks/
		INCLUDES=${SDK_ROOT}/usr/include/
		LIBRARIES=${SDK_ROOT}/usr/lib/

		clang -I${INCLUDES} -F${FRAMEWORKS} -L${LIBRARIES} -Os -dynamiclib -isysroot ${SDK_ROOT} -arch ${ARCH} -fobjc-arc ${IOS_VERSION_MIN_FLAG}=${IOS_VERSION_MIN} -framework Foundation -framework UIKit -framework CoreGraphics -framework QuartzCore -framework ImageIO -lz -lsqlite3 ${INPUT} -o ${DIR}/${BIN_NAME}
done

echo "Creating universal binary..."
FAT_BIN_DIR="bin/universal"
mkdir -p ${FAT_BIN_DIR}
lipo -create bin/**/${BIN_NAME} -output ${FAT_BIN_DIR}/${BIN_NAME}

echo "Copying dylib..."
DYLIB_PATH="./layout/Library/Application Support/FLEXLoader/"
if [ ! -d "$DYLIB_PATH" ]; then
	mkdir -p ./layout/Library/Application\ Support/FLEXLoader/
fi

cp -f bin/universal/libFLEX.dylib layout/Library/Application\ Support/FLEXLoader

echo "Done."
```

这个先使用 git 的 submodule 下载 FLEX 源码，然后使用 clang 进行编译，最后再把生成的 dylib 拷贝到指定的目录中。

## 编写 Tweak

做完前面两步，准备工具就已经做完了，可以开始编写 Tweak 了。

Tweak 工程的创建跟之前一样，只有一个地方需要注意，由于我们是需要在所有的 App 都能使用到这个 Tweak，因此需要指定 `MobileSubstrate Bundle filter` 为 `com.apple.UIKit`。

Tweak.xm 文件的内容如下：

```objc
#include <UIKit/UIKit.h>
#include <objc/runtime.h>
#include <dlfcn.h>

@interface FLEXManager

+ (instancetype)sharedManager;
- (void)showExplorer;

@end


@interface FLEXLoader: NSObject
@end

@implementation FLEXLoader

+ (instancetype)sharedInstance {
	static dispatch_once_t onceToken;
	static FLEXLoader *loader;
	dispatch_once(&onceToken, ^{
		loader = [[FLEXLoader alloc] init];
	});	

	return loader;
}

- (void)show {
	[[objc_getClass("FLEXManager") sharedManager] showExplorer];
}

@end

%ctor {
	NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];

	NSDictionary *pref = [NSDictionary dictionaryWithContentsOfFile:@"/var/mobile/Library/Preferences/com.swiftyper.flexloader.plist"];
	NSString *dylibPath = @"/Library/Application Support/FLEXLoader/libFLEX.dylib";

	if (![[NSFileManager defaultManager] fileExistsAtPath:dylibPath]) {
		NSLog(@"FLEXLoader dylib file not found: %@", dylibPath);
		return;
	} 

	NSString *keyPath = [NSString stringWithFormat:@"FLEXLoaderEnabled-%@", [[NSBundle mainBundle] bundleIdentifier]];
	if ([[pref objectForKey:keyPath] boolValue]) {
		void *handle = dlopen([dylibPath UTF8String], RTLD_NOW);
		if (handle == NULL) {
			char *error = dlerror();
			return;
		} 

		[[NSNotificationCenter defaultCenter] addObserver:[FLEXLoader sharedInstance]
										   selector:@selector(show)
											   name:UIApplicationDidBecomeActiveNotification
											 object:nil];
	}	

	[pool drain];
}
```

代码中我们读取了前面指定的系统配置保存文件，同时使用应用的 `bundleid` 配合前缀判断该应用是否在设置中打开开关。如果有的话，就使用 `dlopen` 函数将动态库加载到应用中。

动态库加载成功后，就可以注册 `UIApplicationDidBecomeActiveNotifcation` 通知，在通知事件中调用 `FLEXManager` 的 `showExplorer` 方法了。

现在还有一个问题，我们如何将 `plist` 文件还有 `dylib` 拷贝到 iOS 设备对应目录中。当然，我们可以使用 `scp` 命令进行拷贝，但是如果每次安装这个 Tweak 还需要先手动拷贝一次也未免太过麻烦。

而 Theos 已经帮我们解决这个问题了，我们只需要在 Tweak 的目录下建立一个 `layout` 目录，这个目录对应的就是 iOS 设备上的根目录，在该 `layout` 目录下所有内容，会在编译 deb 的时候放到设备对应的目录下。

## 效果

至此，我们的 Tweak 就完成了，来看看使用的效果吧。

![inspect-third-party-app-using-flexloader-pic-2](http://7xqonv.com1.z0.glb.clouddn.com/inspect-third-party-app-using-flexloader-pic-2-1.png)

现在，有了这个神器，接下来可以做什么，大家就自便咯。

## 总结

可以在 [Github](https://github.com/buginux/FLEXLoader) 上找到本文章的完整项目源码，如果对大家有帮助的话，可以点个 ★star 哦。

同时，本项目也已经提交到 Cydia 审核，审核通过后就可以直接在 Cydia 市场中搜索 FLEXLoader 进行下载安装。

## 参考资料

* 关于 PreferenceLoader 的更详细介绍可以[参考这里](http://iphonedevwiki.net/index.php/PreferenceLoader)。
* 关于 AppList 的更详细介绍可以[参考这里](http://iphonedevwiki.net/index.php/AppList)。
* AppList 项目地址：https://github.com/rpetrich/AppList
* RevealLoader 项目地址：https://github.com/heardrwt/RevealLoader

## 关注

如果你喜欢这篇文章，可以关注我的公众号，随时获取我最新的博客文章。

![qrcode_for_gh_6e8bddcdfca3_430.jpg](http://upload-images.jianshu.io/upload_images/650096-9155581b667f64b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

