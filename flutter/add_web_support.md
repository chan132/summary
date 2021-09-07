# Add web support to an existing project.

## 背景介绍

* 对`flutter 2.0`之前版本的flutter项目，升级到`flutter 2.0`，然后添加Web支持
* 以下场景以`flutter 1.22.2`升级到`flutter 2.2.3`为例

### 说明

* 以下个别问题是从我个人项目角度出发，如有疑问可随时提issue
* 我个人项目是将已有app转成web项目，让后放至微信公众号上运行
* 以下个别观点仅代表我个人看法，仅供参考



## Upgrade to flutter2

### 升级本地Flutter SDK

1. 切换到stable：`flutter channel stable`

2. 升级到最新版本：`flutter upgrade`

3. 更改pubspec.yaml环境变量中Dart SDK和Flutter对应的版本

### 项目支持空安全

1. 依赖升级到空安全

   1. 详见：[迁移至空安全](https://dart.cn/null-safety/migration-guide)

   2. 检查依赖是否支持空安全：`dart pub outdated --mode=null-safety`【如果有未支持的依赖，则需要联系依赖开发者进行支持】

   3. 将所有依赖升级到支持空安全的版本

      * 更改pubspec.yaml文件：`dart pub upgrade --null-safety`
      * 拉取pub最新依赖：`dart pub upgrade`

   4. 将项目代码支持空安全

      * 使用迁移工具操作

        * 按照以下标记，在对应代码位置添加注释

          ![提示标记](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_aws_1.png)

        * 使用dart migrate 命令启动迁移工具，在浏览器中打开命令行输出的链接即可

      * 手动使用`?、!、required、late`关键字进行更改

2. 分析项目是否完全支持空安全

   1. 分析代码：`flutter analyze`
   
   2. 按照命令行结果的提示修改对应代码
   



## Add web support

### 向现有项目添加Web支持

1. 详见：[使用Flutter构建Web应用](https://flutter.cn/docs/get-started/web)

2. 添加Web支持：`flutter create .`【在项目目录下执行】

   * 如果提示`Ambiguous organization in existing files: {com.mishu, com.mida}. The --org command line argument must be specified to recreate project.`
   * 则通过`flutter create --org <package_name> .`创建制定包名的web项目
   * 由于是通过指定包名的方式创建，所以需要移除Android、iOS项目下新增的文件

3. 通过`flutter devices`命令确认有Chrome，以此确认对web的支持

4. 在所有判断Platform的地方新增对web的判断

   * 引入kIsWeb：`import 'package:flutter/foundation.dart' show kIsWeb;`
   * 判断Platform的地方新增kIsWeb的条件判断【注意：一定先判断kIsWeb，再判断Platform】

5. 适配不支持Web的插件

   1. 通过dart pub插件仓库搜索项目中所使用到的插件，筛选出不支持Web的插件
      * 在generated_plugin_registrant.dart文件中也会有支持web的插件，可以用来辅助筛选
   2. 对于不支持web的插件，通过以下方式进行适配
      * 通过不同的平台引入不同的文件
        * [跨Flutter和Web的条件导入](https://medium.com/flutter-community/conditional-imports-across-flutter-and-web-4b88885a886e)
        * [Dart中的条件引入](https://medium.com/@dvargahali/dart-2-conditional-imports-update-16147a776aa8)
      * 在Web平台使用JS进行实现
        * [在Flutter Web中使用JavaScript](https://medium.com/flutter-community/using-javascript-code-in-flutter-web-903de54a2000)
        * [JS Bundler用例](https://github.com/Vanethos/flutter_dart_js/blob/master/lib/bundler/javascript_bundler.dart)
        * [Dart中的external关键字](https://stackoverflow.com/questions/24929659/what-does-external-mean-in-dart)

### 运行Web应用

1. 直接在本地Chrome中运行：`flutter run -d chorme`

2. 使用IP:Port构建：`flutter run -d chrome --web-hostname <本机IP> --web-port <待访问端口>`【可在局域网的其他设备浏览器，通过IP:Port访问项目】

### 运行后遇到的问题

1. 遇到`MissingPluginException`异常，该异常会提示是哪一个插件不支持Web

   ![MissingPluginException](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_aws_2.png)

   * 解决方案
     * 先找到用到该插件方法的位置，然后先暂时注释，或通过`kIsWeb`进行条件判断然后跳过
     * 后续尽快通过Web适配解决该问题

2. 遇到`Unsupported operation: Platform._operatingSystem`异常，则是在未现判断`kIsWeb`的情况下，使用了`Platform`

   ![MissingPluginException](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_aws_3.png)

   * 解决方案
     * 定位出现异常的位置，遇到该异常通常会在下方提示具体在项目何处调用
     * 在其前方新增`kIsWeb`判断

3. 遇到网络请求`XMLHttpRequest error.`异常，通常是出现了CORS跨域问题

   ![MissingPluginException](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_aws_4.png)

   * 解决方案

     1. Flutter issue参考：[issue 46904](https://github.com/flutter/flutter/issues/46904)

     2. 具体解决方案参考该issue的评论：[issue 评论](https://github.com/flutter/flutter/issues/46904#issuecomment-653088177)

     3. 详细步骤

        * 进入Google Chrome文件夹下：`cd /Applications/Google\ Chrome.app/Contents/MacOS`

        * 新建google-chrome-unsafe.sh文件

          * 创建文件：`touch ./google-chrome-unsafe.sh`

          * 给文件放入“关闭安全检测”的配置：

            `\#!/bin/sh`

            `/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --disable-web-security --user-data-dir="/tmp/unsafe_chrome" $*`

            * 注意：后面的$*是必须要的；其中`/tmp/unsafe_chrome`是用于存放临时文件的，因为打开chrome后会生成一堆文件需要存放

          * 给文件添加执行权限：`chmod a+x google-chrome-unsafe.sh`

          * 给用户的bash_profile添加chrome环境变量：【Flutter会使用这个环境变量来启动chrome】

            * 打开用户的bash_profile文件：`vim ~/.bash_profile`
            * 添加chrome环境变量：`export CHROME_EXECUTABLE="/Applications/Google Chrome.app/Contents/MacOS/google-chrome-unsafe.sh"`

          * 应用环境变量：`source ~/.bash_profile`

          * 最用通过`flutter run -d chrome`验证即可

     4. 附加`Chrome如何关闭安全检测`

        * 给Chrome安装Access-Control-Allow-Origin插件，通过插件开关一键控制【此方案未通过验证】
        * 通过打开一个关闭安全检测的浏览器，打开Flutter Web页面【此方案通过验证，可行】
          * 通过命令`open -n -a "Google Chrome" --args --disable-web-security --user-data-dir=<temp save path>`打开一个浏览器，然后在浏览器中打开Flutter Web页面【temp save path存放临时chrome的位置，最好是一个单独的文件夹，不然后面删除文件的时候难受】
          * 参考链接1：[Overcome CORS跨域](https://medium.com/flutter-community/flutter-web-for-an-enterprise-app-a056fb4e26d1)
          * 参考链接2：[stackoverflow 评论](https://stackoverflow.com/questions/3102819/disable-same-origin-policy-in-chrome)

### 构建Web应用

1. 普通构建：`flutter build web —release`

2. web-renderer为html：`flutter build web --release --web-renderer=html`

3. web-renderer为canvaskit：`flutter build web --release --web-renderer=canvaskit`



## Flutter Web应用优化点

1. 在浏览器中加载时，加载canvaskit.js太慢，这是由于该文件的地址在国外。解决方案如下：

   * 参考文章：[设置镜像地址](https://blog.csdn.net/qq_35867494/article/details/118516893)
   * 构建时配置canvaskit.js的下载地址：`flutter build web --release --web-renderer=canvaskit --dart-define=FLUTTER_WEB_CANVASKIT_URL=https://cdn.jsdelivr.net/npm/canvaskit-wasm@<版本号>/bin/`

2. 在Android微信中打开页面空白，有以下两种情况：

   1. 是由于Android微信的浏览器内核为Tencent X5的原因。解决方案如下：
      * 参考文章：[在微信中显示空白页](https://segmentfault.com/q/1010000040049265?utm_source=tag-newest)
      * 在构建时选择web-renderer为html：`flutter build web --release --web-renderer=html`
   2. 如果是Web worker正在安装，所以只需要等待激活即可。解决方案如下：
      * 参考文章评论：[flutter web运行显示空白页](https://blog.csdn.net/z2008q/article/details/113942549)
      * 将index.html中`waitForActivation(reg.installing ?? reg.waiting);`的`??`替换为`||`，如：`waitForActivation(reg.installing || reg.waiting);`
      * 最后构建时web-renderer选择canvaskit或html均可

3. main.dart.js文件过大。解决方案如下：

   * 参考issue中的解决方案：[issue 46589](https://github.com/flutter/flutter/issues/46589)
   * 可通过gzip在服务端进行压缩，浏览器会自动解压

4. main.dart.js在更新版本后，本地浏览器未更新；这是由于本地缓存问题。解决方案如下：

   * 参考issue中的解决方案：[issue 64968](https://github.com/flutter/flutter/issues/64968)
   * 可通过自动化构建工具，在构建结束时通过在文件名中添加hash值，同时更改index.html中的引用命即可

5. 构建时web-renderer选择canvaskit还是html

   1. 选择html时，会遇到页面中过小的字体会被“截掉”，导致显示不全。【⚠️目前暂未找到合适的解决方案】
      * 参考issue1：[issue 65526](https://github.com/flutter/flutter/issues/65526)
      * 参考issue2：[issue 64904](https://github.com/flutter/flutter/issues/64904)
      * 参考issue3：[issue 63467](https://github.com/flutter/flutter/issues/63467)
      * 尝试使用代码进行测试，发现只要在页面中使用Containerb包裹Text，Text始终会存在问题，如果不进行包裹则不会。但此测试在BottomNavigationBar上又未得到验证，怀疑跟Theme在不同位置使用的DefaultTextStyle不一致，但也未找到在页面中具体使用的是那种DefaultTextStyle
   2. 选择canvaskit时，会遇到以下几个问题
      * 首次加载特别漫长，大概需要15s～16s。主要是被消耗在以下几个地方：
        * 下载canvaskit.js文件（大约3M），从“[unpkg.com](http://unpkg.com)”下载需要2s~3s。【⚠️尝试通过构建时配置FLUTTER_WEB_CANVASKIT_URL为CDN地址，但是始终未替换成功，从浏览器中查看到的请求头还是[unpkg.com](http://unpkg.com)】
        * 由于项目中含有字体文件，所以在首次加载会将字体文件全部下载下来，一个字体文件(大约11M)大概需要5s~6s。解决方案如下：
          * 参考文章：[通过fontTools精简字体](https://cyh.me/2020/04/font-minification/)
          * 常用汉字、字母、标点符号参考：[常用汉字、字符参考](https://tun6.com/font_subset/)
          * 通过工具精简字体文件中的字体，保留常用的中文字体、字母、标点符号即可，即可将字体文件大小缩小，从而使得加载时间缩短【⚠️尝试将项目中的字体文件删除，不再使用自定义字体，但是在首次加载的时候，还是会从“[fonts.gstatic.com](http://fonts.gstatic.com)”下载默认字体（时长并不会减短），就会导致页面首先展示乱码，然后恢复正常】
      * 首次加载后，再次打开时还是会耗时3s～4s左右。【⚠️经过分析后发现，首次加载后浏览器会将canvaskit.js、字体文件等文件缓存，再次打开时会从本地缓存中读取。但由于项目是放在公众号里进行加载，所以存在授权，而授权会消耗1s～2s左右，然后页面开始渲染各个元素，直至页面渲染完成，会消耗3s~4s左右（可以在授权后添加loading，直到flutter中main()函数执行后关闭）】
      * 在微信公众号里面进入应用后，在有列表的页面多次滚动后，页面就会重新加载。在重新加载完成后再次滚动列表，则会出现反复加载页面的问题【⚠️目前不清楚原因，但看上去的感觉有点像内存溢出导致】

6. 【总结】构建时web-renderer选择canvaskit时

   1. 通过fontTools对字体文件提取后，然后对main.dart.js等超过500kB的文件使用gzip，可以保证在首次加载在4s左右，第二次加载时2s左右
   2. 但是项目中所使用到的ListView会存在一个问题，就是在性能较差的手机上会存在严重卡顿的问题
      * 尝试单独写一个列表页面，在性能较差的手机中微信或浏览器，均会存在卡顿的问题
      * 在flutter的issue中也看到了相关问题，至今未关闭
        * 未关闭issue：[issue 52207](https://github.com/flutter/flutter/issues/52207)
        * 被合并的issue：[issue 22314](https://github.com/flutter/flutter/issues/22314)
        * 被合并的issue：[issue 42987](https://github.com/flutter/flutter/issues/42987)
      * 目前尝试按照issue中的相关解决方案进行解决，均未得到较好的效果
   3. 此类卡顿的问题在web-renderer选择html时均无，但是html的方式会遇到字体小于12号时出现被截取的问题，而该问题也在issue中有提到，且目前也未解决
   4. 同时在列表页面，在性能较差的手机（微信）上，会存在滑动过多导致页面被重新加载的问题【性能较好的手机则不会出现该问题】
   5. 所以目前暂时没有一个较好的方案，可以让手机web上的体验得到提升

   **至此，flutter项目目前对于移动端能够有一个很好的体验，但对于web（特别受性能较差手机上的web）体验不够友好**

