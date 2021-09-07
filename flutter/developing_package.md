# Developing Flutter packages.
### 参考文章

* Flutter package的开发与提交：[developing-packages](https://flutter.cn/docs/development/packages-and-plugins/developing-packages)
* Dart package的发布：[publishing-is-forever](https://dart.cn/tools/pub/publishing#publishing-is-forever)
* 编写package的介绍页面：[writing-package-pages](https://dart.cn/guides/libraries/writing-package-pages)



### 提交项目到GitHub

* 在GitHub上创建仓库：[Create a new repository](https://github.com/new)

* 将本地代码提交到仓库

  1. 完成本地项目Git初始化，依次执行以下代码

     $ `git init`

     $ `git add .`

     $ `git commit -m "first commit"`

     $ `git remote add <仓库别名> <仓库https地址>`

  2. 将本地代码提交到Git仓库

     $ `git push --set-upstream <仓库别名> master`

     * 如遇到`fatal: unable to access 'https://github.com/***.git/': LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to github.com:443 `
       * 问题原因：由于Terminal未开代理导致
       * 解决方案：`export ALL_PROXY=socks5://127.0.0.1:1080`
     * 如遇到：输入账号密码后提示`Support for password authentication was removed on August 13, 2021. Please use a personal access token instead.`，同时提示身份验证失败
       * 问题原因：GitHub从2021年8月13号之后，就只能通过个人访问令牌进行身份验证了
       * 详见：[Token authentication requirements for Git operations](https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/)
       * 解决方案参照：[Creating a personal access token](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token)
       * 在身份验证时，使用令牌代替密码即可



### 将开发好的package提交到pub.dev

1. 发布之前确认以下几点

   * pubspec.yaml中的内容完整，查看homepage是否填写【填写项目的GitHub主页】
   * README.md是否编写完整
     * README规范：[README Checklist](https://github.com/ddbeck/readme-checklist/blob/main/checklist.md)
     * markdown基础语法：[commonmark.org](https://commonmark.org/help/)
     * 推荐编辑器：[Typora](https://typora.io/)
   * LICENSE文件是否编写完整，查看year和fullname是否更改
     * 开源LICENSE和标准：[Licenses & Standards](https://opensource.org/licenses)
     * 选择开源LICENSE：[Choose an open source license](https://choosealicense.com/)
   * CHANGLOG.md是否编写版本描述

2. 通过`flutter pub publish --dry-run`命令，检查代码是否通过了分析

   * 出现提示`Package has 0 warnings. `则无问题
   * 如有问题，根据提示进行修改即可

3. 通过`flutter pub publish`命令，将package发布到pub.dev

   * 再次确认：是否准备好上传package，直接输入y即可

     `Looks great! Are you ready to upload your package (y/n)? y `

   * 授权gmail账号访问pub.dev：需要在浏览器中打开改链接，选择对应的gmail账号授权即可

     `Pub needs your authorization to upload packages on your behalf. In a web browser, go to https://accounts.google.com/o/oauth2/auth?*******userinfo.email `

     `Then click "Allow access".`

     `Waiting for your authorization...`

   * 已收到授权，正在处理…

     `Authorization received, processing...`

   * 授权成功，上传package

     `Successfully authorized.`

     `Uploading…`

     `Successfully uploaded package.`

   * 授权失败，提示信息如下

     `It looks like accounts.google.com is having some trouble. `

     `Pub will wait for a while before trying to connect again. `

     `OS Error: Operation timed out, errno = 60, address = accounts.google.com, port = 53481`

     `pub finished with exit code 69`

     * 问题原因：由于Terminal未开代理导致

     * 参考链接：[issue 17070](https://github.com/flutter/flutter/issues/17070)

     * 解决方案：开启Terminal代理

       $ `export HTTP_PROXY=http://proxyAddress:port`

       $ `export HTTPS_PROXY=http://proxyAddress:port`

       * 注意，访问google相关网站时
         * 只能通过HTTP_PROXY和HTTPS_PROXY进行配置，使用ALL_PROXY不会认
         * 只能通过http代理方式访问，socks5代理不会认

4. 上传成功后，稍隔一小会儿，即可在pub.dev上搜索到package了：[pub.dev](https://pub.dev/)

   * ⚠️Tips：给代码添加注释、example有助于提高PUB POINTS



### My packages

* [draggable_float_widget](https://pub.dev/packages/draggable_float_widget)
* [separate_phone_number_input](https://pub.dev/packages/separate_phone_number_input)
* 😊两个非常简单的插件，如果对你有帮助的话，希望不要吝啬你的like和star
* ⌨️如有疑问可以随时提issue给我🙏

