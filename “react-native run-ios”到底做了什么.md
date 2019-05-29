
转载：[地址](https://www.jianshu.com/p/088be846270d)

我们都知道React Navite在开发的时候，需要在React Native根目录下运行react-native run-ios(或run-android)，或者在Xcode中运行原生iOS项目（对于Android则是在Android Studio中运行原生Android项目），然后在对应的React Native根目录下运行npm start(开启nodejs服务，开启JS Server)。
写这篇文章的目的

1、梳理react-native run-ios(android)的完整流程，并识别ios和android的区别。  
2、理解debug下ios的诡异现象，在没开JS Server时，有时加载条为什么会变成load pre bundle并能正常运行，有时为什么闪崩。  
3、帮助不懂原生的朋友快速进入code状态，不要每次都等待react-native run-ios(android)  
4、最后的黑魔法，在团队协作下，写js的可以根本不需要iOS和Android环境（当然，这需要原生开发伙伴的帮助），原生也不需要装nodejs。  

探索“react-native run-ios”到底做了什么

react-native run-ios:  
在控制台可以看到输出：  
```
> $ react-native run-ios
Found Xcode project LayoutDemo.xcodeproj
We couldn't boot your defined simulator due to an already booted simulator. We are limited to one simulator lau
nched at a time.
Launching iPhone 6 (iOS 10.3)...
Building using "xcodebuild -project LayoutDemo.xcodeproj -configuration Debug -scheme LayoutDemo -destination i
d=BB4E36F2-D6B3-447F-91E9-8D1F5B56022E -derivedDataPath build"
1、使用xcodebuild来编译项目
在输出中可以看到React Native首先是寻找Xcode project（寻找方式后面会说明），然后使用 xcodebuild来编译项目。你可以想象就是用xcode打开项目，然后按运行，一样的效果。
2、定义了打bundle包的脚本
在编译参数中设置了React Native的编译脚本（可以在project.pbxproj中查看到react-native-xcode.sh脚本的设置），目录是在：/node_modules/react-native/packager/react-native-xcode.sh。
react-native-xcode.sh：
case "$CONFIGURATION" in
  Debug)  //在Debug模式下
    # Speed up build times by skipping the creation of the offline package for debug
    # builds on the simulator since the packager is supposed to be running anyways.
    if [[ "$PLATFORM_NAME" == *simulator ]]; then
      echo "Skipping bundling for Simulator platform"  //使用模拟器跑时不打bundle包，退出sh脚本
      exit 0;
    fi

    DEV=true //设置DEV为true
    ;;
  "")
    echo "$0 must be invoked by Xcode" 
    exit 1
    ;;
  *)
    DEV=false //设置DEV为false
    ;;
esac
...
# Xcode project file for React Native apps is located in ios/ subfolder
cd ${REACT_NATIVE_DIR}/../.. //进入React Native根目录
...
if [[ "$CONFIGURATION" = "Debug" && ! "$PLATFORM_NAME" == *simulator ]]; then
  ...
  //如果在Debug环境在，并且不是由模拟器运行则对localhost和ip.txt中的内容设置ATS为允许http（因为node.js服务是开在http://localhost:8081上的）
 NSAppTransportSecurity:NSExceptionDomains:localhost:NSTemporaryExceptionAllowsInsecureHTTPLoads bool true" "$PLIST"
  $PLISTBUDDY -c "Add NSAppTransportSecurity:NSExceptionDomains:$IP.xip.io:NSTemporaryExceptionAllowsInsecureHTTPLoads bool true" "$PLIST"
  echo "$IP.xip.io" > "$DEST/ip.txt"
fi
...
BUNDLE_FILE="$DEST/main.jsbundle"  //设置输出main.jsbundle的目录
//运行react-native下的bundle命令，相当于自己运行`react-native bundle`
$NODE_BINARY "$REACT_NATIVE_DIR/local-cli/cli.js" bundle \
  --entry-file "$ENTRY_FILE" \
  --platform ios \
  --dev $DEV \
  --reset-cache \
  --bundle-output "$BUNDLE_FILE" \
  --assets-dest "$DEST"
```
从脚本能看出在Debug模式下，不会为模拟器打bundle包，但是会为真机打bundle包，这也就是为什么我们在真机调试后，然后断开nodejs服务（不能打开remote debug js模式，不然会闪崩），重新进入应用时也会正确的加载bundle，屏幕上方会出现'load pre bundle'的字样，并且是黑色的背景条，如果是加载nodejs服务器的则会是绿色的背景条并出现加载进度。但是当我们在模拟器上做同样的操作时，比如先正常打开nodejs服务器加载调试，然后在重新打开应用，它加载不到bundle会崩溃。
当我们设置Rlease模式时，必须用xcode编译才会打bundle包，此时无论是在模拟器还是在真机运行，都会打bundle包。  
输出如下：  
```
/Users/lyxia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Intermediates/LayoutDemo.build/Debug-iphoneos/LayoutDemo.build/Script-00DD1BFF1BD5951E006B06BC.sh
+DEST=/Users/lyxia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Products/Debug-iphoneos/LayoutDemo.app
+ [[ Debug = \D\e\b\u\g ]]
+ [[ ! iphoneos == *simulator ]]
+ PLISTBUDDY=/usr/libexec/PlistBuddy

+PLIST=/Users/lyxia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Products/Debug-iphoneos/LayoutDemo.app/Info.plist
++ ipconfig getifaddr en0
+ IP=192.168.16.111
+ '[' -z 192.168.16.111 ']'
+ /usr/libexec/PlistBuddy -c 'Add NSAppTransportSecurity:NSExceptionDomains:localhost:NSTemporaryExceptionAllow
sInsecureHTTPLoads bool true' /Users/lyxia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Products/Debug
-iphoneos/LayoutDemo.app/Info.plist
+ /usr/libexec/PlistBuddy -c 'Add NSAppTransportSecurity:NSExceptionDomains:192.168.16.111.xip.io:NSTemporaryEx
ceptionAllowsInsecureHTTPLoads bool true' /Users/lyxia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Pr
oducts/Debug-iphoneos/LayoutDemo.app/Info.plist
+ echo 192.168.16.111.xip.io
+BUNDLE_FILE=/Users/lyxia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Products/Debug-iphoneos/LayoutDemo.app/main.jsbundle

+ node /Users/lyxia/Documents/ios/React_Native/LayoutDemo/node_modules/react-native/local-cli/cli.js bundle --entry-file index.ios.js --platform ios --dev true --reset-cache --bundle-output/Users/lyxia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Products/Debug-iphoneos/LayoutDemo.app/main.jsbundle --assets-dest /Users/ly
xia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Products/Debug-iphoneos/LayoutDemo.app

[2017-04-01 11:26:20] <START> Initializing Packager
[2017-04-01 11:26:21] <START> Building Haste Map
[2017-04-01 11:26:21] <END>   Building Haste Map (575ms)
[2017-04-01 11:26:21] <END>   Initializing Packager (1644ms)
[2017-04-01 11:26:21] <START> Transforming files

Warning: The transform cache was reset.

[2017-04-01 11:26:40] <END>   Transforming files (18750ms)
bundle: start
bundle: finish
bundle: Writing bundle output to: /Users/lyxia/Documents/ios/React_Native/LayoutDemo/ios/build/Build/Products/D
ebug-iphoneos/LayoutDemo.app/main.jsbundle
bundle: Copying 5 asset files
bundle: Done writing bundle output
bundle: Done copying assets
+ [[ ! -n true ]]
```
3、自动弹出一个框来启动nodejs服务  
在../node_modules/react-native/React/React.xcodeproj/project.pbxproj文件中我们可以看到PBXShellScriptBuildPhase section中定义了Start Packager的shell执行，最终会执行到../node_modules/react-native/packager/launchPackager.command。一路追溯，可以看到最后执行到node "$THIS_DIR/../local-cli/cli.js" start "$@"也就是我们常用的npm start。
总结：所以现在我们整理一下这整个流程：  

使用xcodebuild编译项目  
因为在使用react-native init <项目名>生成的项目中，project.pbxproj里面会添加生成bundle的Bundle React Native code and images编译参数，所以我们在真机完成调试后，nodejs关了，也能去加载本地的bundle，因为它已经帮我们打包好了。
因为在React的project.pbxproj中添加了Start Packager的编译参数，所以它会判断是否已经开启nodejs服务，如果没有则会帮我们开启。  

组合：按需求组合使用  
这个流程使我们只要运行react-native run-ios就可以编译项目，开启nodejs服务。  
因此我们可以换个方向来想，如果我们是在原生的iOS项目中接入RN，那应该怎么做呢？  
只需用Xcode运行原生项目，然后npm start即可。当然我们也可以在编译参数中加入Bundle React Native code and images让它每次执行为我们自动打包最新的bundle，这样当我们调试好后，就算关了nodejs服务，也能继续运行。
问题：如何寻找.xcodeproj文件  
在../node_modules/react-native/local-cli/runIOS/runIOS.js里面可以找到react-native run-ios的实现，并且有各种参数的举例说明：  
```
{
    desc: 'Pass a non-standard location of iOS directory',
    cmd: 'react-native run-ios --project-path "./app/ios"',
}
...
{
    command: '--project-path [string]',
    description: 'Path relative to project root where the Xcode project '
      + '(.xcodeproj) lives. The default is \'ios\'.',
    default: 'ios',
  }
```
可以看到默认是在\ios.目录下寻找以.xcworkspace或者.xcodeproj结尾的文件，不过可以使用--project-path [string]来指定文件目录。  
探索“react-native run-android”到底做了什么  

同样我们在控制台运行react-native run-android，看输出：  
```
> $ react-native run-android
Starting JS server...
Running /Users/lyxia/Documents/Android/adt-bundle-mac-x86_64-20140702/sdk//platform-tools/adb -s C
oolpad5890-a1778fd9 reverse tcp:8081 tcp:8081
Building and installing the app on the device (cd android && ./gradlew installDebug)...
Starting a new Gradle Daemon for this build (subsequent builds will be faster).
> Configuring > 1/2 projects > :app > Compiling script into cache^C
:app:preBuild UP-TO-DATE
:app:preDebugBuild UP-TO-DATE
:app:checkDebugManifest
:app:preReleaseBuild UP-TO-DATE
:app:prepareComAndroidSupportAppcompatV72301Library
:app:prepareComAndroidSupportRecyclerviewV72301Library
:app:prepareComAndroidSupportSupportV42321Library
```
是的，Android和我们的iOS宝宝是不一样的，他一开始就去查看JS Server是否开启，如果没开他就会去开启，然后使用gradle来编译项目并安装。  
可以在../node_modules/react-native/local-cli/runAndroid/runAndroid.js中查看react-native run-android的实现，这里不贴出来了。  

注意点  
1、在Release下，会打包bundle：  
```
if (args.configuration.toUpperCase() === 'RELEASE') {
      console.log(chalk.bold(
        'Generating the bundle for the release build...'
      ));
      child_process.execSync(
        'react-native bundle ' +
        '--platform android ' +
        '--dev false ' +
        '--entry-file index.android.js ' +
        `--bundle-output ${androidProjectDir}/app/src/main/assets/index.android.bundle ` +
        `--assets-dest ${androidProjectDir}/app/src/main/res/`,
        {
          stdio: [process.stdin, process.stdout, process.stderr],
        }
      );
```
2、如何验证android项目是否存在：  
在React Native根目录下寻找android/gradlew，是的，你没看错，全程不可配，乖乖放好路径吧，不然就使用Android Studio运行，然后执行npm start。  
总结：整理react-native run-android的完整流程：  

检查是否需要开启JS Server  
检查是否存在android/gradlew  
检查是否有可用的Android设备已连接  
检查是否在release环境下，如果是则打包bundle  
使用gradle编译android项目并安装  

条理比iOS清晰许多，因为都写在一个文件中。  
对比“react-native run-ios(android)的相同与不同”  

相同点：  

都是要去寻找原生项目是否存在，iOS是存在/ios/*.xcworkspace或者/ios/.xcodeproj结尾的文件，Android是存在android/gradlew文件。  
都会为我们开启JS Server。  
都会在Release环境下为我们打包Bundle。  
都可以真机运行。   
 
不同点：  

iOS的项目路径可配，当我们在原生的iOS中接入React Native时，项目路径很可能不符合它默认的路径，此时可以通过--project-path修改。而Android不可以指定项目路径，如果/android/gradlew不存在，则不可以使用react-native run-android

Debug环境下，iOS在真机上运行时（react-native run-ios --device），也会我们打包bundle，但是Android不会。  
iOS使用xcodebuild来编译项目，Android使用gradle来编译。  

使用以上知识来解决常见问题  

问题1：  
我需不需要每天开机都react-native run-ios(android)  
回答：  
在没有更改原生代码的情况下，是不需要的，只要设备或者是模拟器上安装了应用，只需要npm start开启JS Server即可，让应用能加载到nodejs服务上的js bundle即可。  
问题2：   
如果我电脑上没有Android和iOS环境，能不能调试RN项目。    
回答：  
可以跑，只需要手机上已经装好RN的Debug版的项目，并且电脑开启了JS Server。Android机需要在开发者选项里把ip和端口改成开了JS Server的ip和端口即可。iOS就要麻烦些了，需要在原生添加ip.txt文件，然后指定ip为开了JS Server的ip，重编，即可（这里就需要有Xcode了）。  
问题3：  
可不可以使用1个JS Server同时调试iOS和Android项目。  
回答：  
可以的。  
