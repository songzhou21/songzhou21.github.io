---
layout: post
title: "iOS 手动创建 Framework"
date: 2017-05-29
categories: programming
---

# iOS 手动创建 Framework

## MyFramework project

1. new project -> Cocoa touch Framework

2. add headers to `MyFramework.h`

```objective-c
// Test.h, Test.m
@interface Test : NSObject

- (NSString* )test;

@end


@implementation Test

- (NSString* )test {
    return @"test success";
}

@end


// MyFramework.h
#import <MyFramework/Test.h>

```

3. add headers to Public

    MyFramework target -> Build Phases -> Headers -> Public

4. build target -> MyFramework output

## Framework test Demo project

1. **copy** MyFramework to project

    using the MyFramework.framework output from last step

2. Link Framework

    target -> Build Phases -> Link Binary With Libraries -> add MyFramework.framework to list

3. copy Framework to app “Framework” destination  
    target -> Build Phases -> New Copy Files Phases
    - Destination -> “Framework”
    - add MyFramework.framework to list

### test

```objective-c
#import <MyFramework/MyFramework.h>


- (void)viewDidLoad {
    [super viewDidLoad];
    
    Test* t = [Test new];
    NSLog(@"%@", [t test]);
    
    // Do any additional setup after loading the view, typically from a nib.
}

```



# 聚合

这是最后一步，上述步骤创建的 Framework 只支持模拟器的指令集，为了能够使 Framework 同时支持真机和模拟器，需要创建一个包含真机指令集和模拟器指令集的 Framework。

指令集表格参考:


```

arm64 = iPhone 6s/6SP, iPhone 6/6P, iPhone 5s, iPad Air, Retina iPad Mini
armv7s = iPhone 5, iPhone 5c, iPad 4
armv7  = iPhone 3GS, iPhone 4, iPhone 4S, iPod 3G/4G/5G, iPad, iPad 2, iPad 3, iPad Mini   
i386 = 32 bit simulator
x86_64 = 64 bit simulator
```



创建支持模拟器和真机的 Framework:

```sh
lipo -create /.../Build/Products/Release-iphonesimulator/MyFramework.framework/MyFramework  /.../Build/Products/Release-iphoneos/MyFramework.framework/MyFramework  -output MyFramework
```



 查看聚集后的 Framework 支持的指令集:

```sh
-> lipo -info MyFramework
-> Architectures in the fat file: MyFramework are: i386 x86_64 armv7 arm64 
```



可以通过创建一个 aggregate 的 target 和脚本来简化这些步骤

Editor -> Add Target -> cross-platform -> Aggregate



新建一个 script，`aggregate.sh`，放在项目根目录

```sh
# Sets the target folders and the final framework product.
# 如果工程名称和Framework的Target名称不一样的话，要自定义FMKNAME
# 例如: FMK_NAME = "MyFramework"
FMK_NAME=${PROJECT_NAME}

# Install dir will be the final output to the framework.
# The following line create it in the root folder of the current project.
INSTALL_DIR=${SRCROOT}/Products/${FMK_NAME}.framework

# Working dir will be deleted after the framework creation.
WRK_DIR=build
DEVICE_DIR=${WRK_DIR}/Release-iphoneos/${FMK_NAME}.framework
SIMULATOR_DIR=${WRK_DIR}/Release-iphonesimulator/${FMK_NAME}.framework

# -configuration ${CONFIGURATION}
# Clean and Building both architectures.
xcodebuild -configuration "Release" -target "${FMK_NAME}" -sdk iphoneos clean build
xcodebuild -configuration "Release" -target "${FMK_NAME}" -sdk iphonesimulator clean build

# Cleaning the oldest.
if [ -d "${INSTALL_DIR}" ]
then
rm -rf "${INSTALL_DIR}"
fi

mkdir -p "${INSTALL_DIR}"
cp -R "${DEVICE_DIR}/" "${INSTALL_DIR}/"

# Uses the Lipo Tool to merge both binary files (i386 + armv6/armv7) into one Universal final product.
lipo -create "${DEVICE_DIR}/${FMK_NAME}" "${SIMULATOR_DIR}/${FMK_NAME}" -output "${INSTALL_DIR}/${FMK_NAME}"

rm -r "${WRK_DIR}"
open "${INSTALL_DIR}"

```

在 aggregate target 的 Build Phases 里的 Run Script 内容中 

`"${SRCROOT}/aggregate.sh"`

最后选择 `Generic iOS Device`， `cmd+B` 就能看到聚集后的 Framework



refs:  
[MyFramework repo](https://bitbucket.org/songzhou/ios-framework)

[MyFramework test demo repo](https://bitbucket.org/songzhou/ios-framework-demo)

[Framework Programming Guide](https://developer.apple.com/library/content/documentation/MacOSX/Conceptual/BPFrameworks/Tasks/CreatingFrameworks.html)

[浅谈iOS中Library和Framework](http://blog.coryphaei.com/2015/12/30/浅谈iOS中Library和Framework/)

[iOS-制作Framework](http://www.jianshu.com/p/ef3d5b7e7006)