play用户docker入门
=============

docker简介
-----------
一句话介绍:可以使用各种命令和restful api控制的超快“虚拟机”。
        
    1. 利用linux的内核技术，实现软虚拟机，仅能够运行基于linux的系统
    2. 运行镜像时,可以通过参数改变虚拟机内部的状态，例如环境变量
    3. 使用类git的方式保存、分发、克隆镜像,使镜像可以像源代码一样push  pull clone 以及继承

这些有趣的特性使软件项目有了非常有趣的玩法:
    1. 开发的时候，镜像直接就包含了操作系统，测试、测试环境、生产环境，环境都是一样的，可以避免没有装xx库之类的低级错误
    2. 使用环境变量区分不同的环境、设置容易改变的参数，需要修改的时候，仅仅需要修改下参数，重启就好了
    3. 可以大规模的批量部署，仅仅是一个命令的事情，也很容易集中管理
    4. 可以平滑升降级线上环境
    5. 可以很简单的组合互相依赖的服务

docker镜像简介
-----------
docker的镜像和其他虚拟机镜像的区别是，它是分层、只读的。但是在镜像被启动运行之后，最上层会附加一层可写的文件系统，运行完毕后会保存为最新的
 一层。类似于git的不同版本一样，历史版本不可修改，但是你可以使用以前的版本创建一个分支 !(图片)[http://udn.yyuap.com/doc/chinese_docker/terms/images/docker-filesystems-multilayer.png]

docker使用Dockerfile来创建镜像，一个web静态网站镜像如下
    ·
    FROM livehl/nginx
    copy ["www/", "/usr/share/nginx/html"]
    EXPOSE 80
    CMD ["/bin/sh","-c", "nginx -g 'daemon off;'"]
    ·
第一行表示继承自哪个镜像。官方已经打包好了各种操作系统以及常用工具包的镜像，范例表示引用了[docker官方镜像服务](https://hub.docker.com/r/livehl/nginx/)的镜像。
第二行表示拷贝Dockerfile同目录下的www目录到镜像的/usr/share/nginx/html目录下。
第三行告诉docker，需要开放80端口。
第四行告诉docker，系统启动后默认执行的命令
构建完毕镜像后，就可以启动了。效果即是部署了一个静态的网站。

docker镜像服务简介
-----------
docker镜像创建完毕之后，仅仅是放在了本地，如果需要给其他人或者机器的话，需要一个类似与github一样的镜像管理平台才行，用法也和github差不多。
    1. 创建一个镜像，这时候什么都没有
    2. 镜像发布
        a. 本地构建，push
        b. 自动构建,从github、bitbucket.org拉取git仓库，然后直接构建镜像
   https://hub.docker.com/r/livehl/cow/  这个镜像是由 https://github.com/livehl/cow   这个git库自动构建的，每当git库有更新时，镜像服务商会自动签出最新的
代码，依据设置自动打包，发布版本
如果在国外使用，推荐用docker官方的[hub.docker.com](hub.docker.com)。国内可以使用阿里云容器和daocloud容器的，速度快。当然项目小还是使用hub.docker.com方便些。
使用镜像仅仅需要docker pull hub.docker.com/livehl/cow 即可下载到本地，对于hub.docker.com ,可以省略前面的域名，仅仅输入docker pull livehl/cow 即可

docker运行简介
-----------
docker运行时的虚拟机叫做容器，使用参数可以覆盖镜像内的环境变量、设置网络、内存cpu限制，还能链接其他容器，对于一般应用而言，基本上只需要设置开放端口号、环境变量、链接其他程序
就够了，而且一般不需要手动运行，有集中的管理平台管理。docker运行的详细介绍看[这里](http://dockone.io/article/152)
docker管理简介
-----------
docker最强大的地方是可以通过restful接口管理一个物理机上的所有容器，于是催生出了各种各样的docker管理平台，甚至还有了*容器即服务(Container as a Service)*的概念。
国内技术比较成熟的有daocloud和阿里云；daocloud侧重于企业市场，企业收费个人免费，但是不交费连push镜像的权限都没有。阿里云侧重企业市场，有很强大的扩展功能，但是商用不久，还是有各种小bug。
推荐使用阿里云的容器服务，反正免费，而且功能强大。[阿里云容器官方文档](https://help.aliyun.com/product/25972.html)

play2配置docker
-----------
这是专门为play2用户快速使用docker入门的，篇幅可能偏长。操作步骤顺序如下
    1. 注册镜像服务，以[阿里云](https://cr.console.aliyun.com/#/docker/image/create)为例,设置仓库密码。在打包环境的docker中执行docker login registry.aliyuncs.com  
    2. 创建新镜像，设置代码源 选择 本地仓库
    3. build.sbt 中添加引用
        `
        import com.typesafe.sbt.packager.docker._
        import NativePackagerHelper._
        `
    4. build.sbt 增加docker插件 ` val root = (project in file(".")).enablePlugins(PlayScala).enablePlugins(DockerPlugin).enablePlugins(JavaAppPackaging) `
    5. build.sbt 增加移除pid设置(如果不设置，容器服务的自动重启特性将会失效)    `javaOptions in Universal ++= Seq("-Dpidfile.path=/dev/null")`
    6. build.sbt 增加以下设置
        `
        dockerBaseImage := "livehl/java8"

        packageName in Docker := packageName.value

        dockerExposedPorts := Seq(9000)

        dockerRepository :=Some("registry.aliyuncs.com/用户名")

        `
        其中用户名替换为镜像服务的用户名
    7. 执行sbt docker:publish  即可将项目发布到镜像服务器

    如果build.sbt中设置 `dockerUpdateLatest in Docker := true` ，则sbt会自动更新latest版本的镜像。配合webhook可以自动更新已经部署的应用

    至此docker镜像构建完了。如果有错误，检查下列问题
    1. 笔者sbt版本 sbt.version=0.13.11  play版本 2.4.x
    2. packageName必须和镜像服务的镜像名字一致
    3. registry.aliyuncs.com/用户名 不需要填写镜像名字  只能是registry.aliyuncs.com/handuser  不能是 registry.aliyuncs.com/handuser/app
    4. 没了，去google你的问题吧。或者来qq群132569382 找子轩


前端后端分离部署
-----------
高级特性-多机房部署
-----------
高级特性-自动重启
-----------
高级特性-自动发布
-----------
