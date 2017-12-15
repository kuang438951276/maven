
# Android 项目部署之Nexus私服搭建和应用

## 一、概述
 &emsp;Nexus是一个基于maven的仓库管理的社区项目.主要的使用场景就是可以在局域网搭建一个maven私服,用来部署第三方公共构件或者作为远程仓库在该局域网的一个代理.简单举几个例子就是:
### 1.第三方Jar包可以放在nexus上,项目可以直接通过Url和路径配置直接引用.方便进行统一管理.
### 2.同时有多个项目在开发的时候,一些共用基础模块可以单独抽取到nexus上,需要用的项目直接从nexus上拉取就行
### 3.一些封闭开发的过程中开发机是不能上公网的,所以连接central repository和下载jar就比较麻烦,这时就可以用nexus搭建起来一个介于公网和局域网之间的桥梁.
## 二、搭建
### 1.下载&配置

&emsp;这里使用的是Nexus OSS开源版,官网下载地址:http://www.sonatype.org/nexus/go/ 

&emsp;把压缩包解压之后进入bin文件夹就可以运行cmd命令来启动nexus,通过查看bin/nexus脚本发现可以修改脚本来适配自己的需求,例如修改Nexus的root路径,如果需要以root身份来启动Nexus就需要设置RUN_AS_USER=root,设置app名字和登陆名字等.也可以去conf/nexus.properties文件修改端口之类的信息. 

&emsp;接下来直接运行Nexus脚本,进入解压之后的文件夹的bin目录下，使用cmd命令nexus start启动服务

![image](https://raw.githubusercontent.com/kuang438951276/framework/master/img/1.png)

### 2.创建仓库

&emsp;从状态上来看nexus已经启动起来了,Nexus启动之后就可以在浏览器访问类似 http://192.168.8.208:9999/nexus/ 地址,其中ip为当前服务器ip,端口Nexus默认为8081,可以在conf/nexus.properties文件中修改端口.进入web管理页面后右上角登陆,Nexus默认的账号密码是admin:admin123,然后点击左侧的repositories查看现有的仓库列表

![image](https://raw.githubusercontent.com/kuang438951276/framework/master/img/2.png)

这里的仓库分了四种类型 

&emsp;1.hosted(宿主仓库):用来部署自己,第三方或者公共仓库的构件 

&emsp;2.proxy(代理仓库):代理远程仓库 

&emsp;3.virtual(虚拟仓库):默认提供了一个 CentralM1虚拟仓库 用来将maven 2适配为maven 1 

&emsp;4.group(仓库组):统一管理多个仓库

&emsp;这里以建立hosted仓库为例简单介绍Nexus在Android开发中的实际使用情况.点击repositories->add 键入ID(部署项目的标识) Name等属性,这里需要注意的是,如果该仓库有多次部署的情况的话,将policy设置为allow redeploy,不然后续在部署的时候会出现403错误.

![image](https://raw.githubusercontent.com/kuang438951276/framework/master/img/3.png)

&emsp;建立好新的仓库之后需要配置一下相关账号信息.在安全选项下选择用户选项,可以看到三个默认的账号,分别是管理员账号,部署账号和Nexus账号.正常访问仓库内容的时候是不需要这三个账户的,一般也就是把部署账号暴露出去,方便仓库项目维护人员部署项目使用.所以这里可以用默认的Deployment账户(记得reset下密码).也可以新建一个账号来使用,新建的时候可以通过add role management来控制该账号的权限.

![image](https://raw.githubusercontent.com/kuang438951276/framework/master/img/4.png)
![image](https://raw.githubusercontent.com/kuang438951276/framework/master/img/5.png)

&emsp;回到repositories选项就可以看到新建出来的仓库,点击仓库URL可以直接进入到仓库路径,当然现在还没有部署项目.到此为止搭建Maven私服Nexus端的配置和部署都已经完成,下面开始Nexus Android端的配置部署和应用.

### 3.Android端私服仓库应用

#### 3.1上传aar包

&emsp;在需要上传aar包的模块中的build.gradle文件中使用uploadArchives方法向指定仓库上传有效的aar文件.并且配置pom文件信息.目前配置是默认走repository中的仓库,如果在版本号后面加上-SNAPSHOT就会走snapshotRepository中的仓库.artifactId是该模块的名字,groupId是仓库包名路径.代码所示：

```
apply plugin: 'maven'
uploadArchives {
    repositories {
        mavenDeployer {
            snapshotRepository(url: project.localMavenUrlSnapshot) {
                authentication(userName: project.localMavenUser, password:  project.localMavenPwd)
            }

            repository(url: project.localMavenUrlRelease) {
                authentication(userName:  project.localMavenUser, password: project.localMavenPwd)
            }

            pom.project {
                version '1.0.1'
                artifactId 'cloudapilib'
                groupId 'com.topband.lib'
                packaging 'aar'
                description 'dependences lib'
            }
        }
    }
}
```

&emsp;在gradle.properties配置地址和用户名及密码

```
localMavenUrlRelease=http://192.168.8.208:9999/nexus/content/repositories/topbandlib/
localMavenUrlSnapshot=http://192.168.8.208:9999/nexus/content/repositories/topbandlib-snapshot/

localMavenUser=kuangyong
localMavenPwd=123456
```

&emsp;在Android Studio的Gradle project插件中选中datetimepicker模块运行uploadArchives任务就可以将该模块的aar文件以1.0.1的版本部署到Nexus上,供其他项目引用.

![image](https://raw.githubusercontent.com/kuang438951276/framework/master/img/6.png)

#### 3.2.aar包引用

首先在project的gradle文件中配置本地maven私服库地址

```
allprojects {
    repositories {
        jcenter()
        mavenLocal()
        mavenCentral()
        //Add the JitPack repository
        maven { url "https://jitpack.io" }
    }
    dependencies{
        repositories {
            maven { url 'http://192.168.8.208:9999/nexus/content/repositories/topbandlib/' }
        }
    }
}
```

&emsp;接着配置module的gradle中配置引用，如：compile 'com.topband.lib:cloudapilib:1.0.1@aar'

```
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    compile 'com.android.support:design:25.1.1'
    //阿里巴巴界面跳转路由
    annotationProcessor "com.alibaba:arouter-compiler:$rootProject.routerVersion"
    compile 'com.alibaba:arouter-api:1.2.2'
    //界面卡顿检测工具
    compile 'com.github.markzhai:blockcanary-android:1.5.0'
    compile 'com.android.support.constraint:constraint-layout:1.0.2'
    //云服务
    compile 'com.android.support:appcompat-v7:26.+'
    compile 'com.squareup.okhttp3:okhttp:3.8.1'
    compile 'com.squareup.okhttp3:logging-interceptor:3.4.1'
    compile 'com.google.code.gson:gson:2.7'
    compile 'org.greenrobot:greendao:3.2.2'
    compile 'com.google.zxing:core:3.2.1'
    compile 'com.jakewharton.hugo:hugo-annotations:1.2.1'
    compile 'com.topband.lib:cloudapilib:1.0.1@aar'
}
```
配置完成之后就可以在代码中调用aar包的API了。

参考文献：http://blog.csdn.net/l2show/article/details/48653949
