---
title: 前端后端一体化自动构建
author: 大大白
date: 2018-03-01 11:48:00
categories:
- 服务端
- 自动构建
tags:
- 前后端一体化构建
---
前端项目每次发布的时候需要手动 run build 然后提交 dist。难道就没有办法来解决这种没意义而又重复劳作的事情吗？git上面居然会有编译后的文件，你的良心不会痛吗？配置文件是通过匹配域名来区分不同的变量，不觉得这很 low 吗？你还在为当接口出问题因为看不到服务端代码而不知道怎么跟人家吵架而感到苦恼吗？用 npm run build 然后提交 dist 发布这样会隐藏着一个定时炸弹。在 windows 下是大小写不敏感的，当你编译完之后那些文件已经互相链接在一起，发布到 linux 当然可以正常运行。但是假如有一天，项目组决定不能再在本地构建然后提交代码了，必须要在云服务器构建。那么 linux 下是大小写敏感的，编译的时候文件链接不到一起，就会报各种 xxx 文件找不到。
如果你曾经遇到过以上几点，那么恭喜你，这篇文章可以为你解决全部问题！
<!-- more -->

## 前置知识点
Maven 是一个项目管理的综合工具。Maven 提供了开发人员构建一个完整的生命周期框架，简化和标准化项目建设过程。MAVEN 定义了`标准的目录结构`。以下是 Maven 定义的标准目录 Standard Directory Layout

|     Directory      |                                Description                                 |
| :----------------: | :------------------------------------------------------------------------: |
|   src/main/java    |                        Application/Library sources                         |
| src/main/resources |                       Application/Library resources                        |
|  src/main/filters  |                           Resource filter files                            |
|  src/main/webapp   |                          Web application sources                           |
|   src/test/java    |                                Test sources                                |
| src/test/resources |                               Test resources                               |
|  src/test/filters  |                         Test resource filter files                         |
|       src/it       |                 Integration Tests (primarily for plugins)                  |
|    src/assembly    |                            Assembly descriptors                            |
|      src/site      |                                    Site                                    |
|    LICENSE.txt     |                             Project's license                              |
|     NOTICE.txt     | Notices and attributions required by libraries that the project depends on |
|     README.txt     |                              Project's readme                              |

在这里我们主要重点关注这三个目录和生命周期中的两个阶段
- 三个目录
  - Application/Library sources 的目录`src/main/java`，本文后续简称为 source。
  - Application/Library resources 的目录`src/main/resources`，本文后续简称为 resource。
  - Web application sources 的目录`src/main/webapp`，本文后续简称为 webResource。
- 两个阶段
  - compile 阶段
  - package 阶段（coping 相关文件，打成 war 包）

简单来说，compile 阶段会编译`src/main/java`目录中的文件，package 阶段要分两个步骤：第一步，复制相关的文件，把刚才 compile 阶段编译好的文件加上`src/main/webapp`和`src/main/resources`的文件复制到一个新的文件夹中一般默认情况下是`program-name-0.1.0-SNAPSHOT`；第二步，把`program-name-0.1.0-SNAPSHOT`打成 war 包`program-name-0.1.0-SNAPSHOT.war`。

至此，前置知识就都介绍完了。接下来我们会在 maven 构建的流程下配置我们前端的构建信息。
我们会把目录结构设置成这个样子
```
 .
 |-- pom.xml
 `-- src
     `-- main
         |-- java
         |-- js
         |-- webapp
```
接下来我们会通过几个问题来对自动化构建抽丝剥茧，最终使你对整个构建有一个全面的认识。

## 思考几个问题
### js 目录能不能放在 webapp 底下？为什么？
- 首先，做过 J2EE 开发的同学一定会觉得页面文件应该放到 webapp 里面。但是事实上 js 目录底下的文件并不是直接就是页面文件，它需要经过编译后才能变成页面文件。所以`页面文件应该放到webapp里面`这句话没有错，但是不够确切，应该这么说会恰当一点`webapp里面放的是可直接访问的页面文件`。
- 然后，说明它不放在 webapp 里面是正确的。这些 js 并不是可以直接访问的页面必须要经过编译后才能被访问，所以不放在 webapp 里面是正确的。
- 再来，说明它放在 webapp 里面是错误的。我们 js 目录底下的文件数量可以说是非常多的，想一下 node_modules 就知道了，即便是不考虑 node_modules 我们项目的文件数量在很多情况下都可以说是成百上千，工程大的更是成千上万，这么多数量的文件放在 webapp 里面，当 maven 在 package 阶段的第一步的时候就会占用大量内存（亲测，云服务器的 web 容器因此挂掉了）造成没必要的资源浪费，而且这样打包时间太长了，把时间都耗费在 IO 上面了。所以它放在 webapp 里面是错误的。
- 最后，总结一下。`webapp底下应该放的是可直接访问的页面`。

### 不同的环境配置了不同的 webpack，如果不能手动 run build，那我该如何让其自动选择对应环境的 webpack？

pom.xml
```xml
    <!-- compile阶段的相关配置 -->
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.1</version>
        <configuration>
            <source>${java.version}</source>
            <target>${java.version}</target>
            <showWarnings>true</showWarnings>
        </configuration>
    </plugin>
    <!-- package阶段之前执行的一些插件 -->
    <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>1.4.0</version>
        <executions>
            <execution>
                <id>cnpm install</id>
                <phase>prepare-package</phase>
                <goals>
                    <goal>exec</goal>
                </goals>
                <configuration>
                    <executable>cnpm</executable>
                    <workingDirectory>./src/main/js</workingDirectory>
                    <arguments>
                        <argument>install</argument>
                    </arguments>
                </configuration>
            </execution>
            <execution>
                <id>gulp build</id>
                <phase>prepare-package</phase>
                <goals>
                    <goal>exec</goal>
                </goals>
                <configuration>
                    <executable>cnpm</executable>
                    <workingDirectory>./src/main/js</workingDirectory>
                    <arguments>
                        <argument>run</argument>
                        <argument>build:${env}</argument>
                    </arguments>
                </configuration>
            </execution>
        </executions>
    </plugin>
    <!-- package阶段的相关配置 -->
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-war-plugin</artifactId>
        <version>2.5</version>
        <configuration>
            <webResources>
                <resource>
                    <directory>src/main/filters/webResources/${env}</directory>
                </resource>
            </webResources>
            <warSourceIncludes>index.html,*.js,WEB-INF/**,static/**</warSourceIncludes>
        </configuration>
    </plugin>
```

package.json
```json
"scripts": {
    "build:development":    "rimraf ../webapp/* && webpack -p --config ./webpack/webpack.development.config.js    --progress --profile --colors --display-error-details --display-cached",
    "build:test":           "rimraf ../webapp/* && webpack -p --config ./webpack/webpack.test.config.js           --progress --profile --colors --display-error-details --display-cached",
    "build:preproduction":  "rimraf ../webapp/* && webpack -p --config ./webpack/webpack.preproduction.config.js  --progress --profile --colors --display-error-details --display-cached",
    "build:product":        "rimraf ../webapp/* && webpack -p --config ./webpack/webpack.product.config.js        --progress --profile --colors --display-error-details --display-cached"
}
```

1. maven 插件[maven-compiler-plugin](http://maven.apache.org/components/plugins/maven-compiler-plugin/)
 maven 编译阶段的配置。`<source />`:源版本，因为有些 web 容器是要指定版本的，这里默认源和目标版本都一致。`<target />`:目标版本，因为有些 web 容器是要指定版本的，这里默认源和目标版本都一致。`<showWarnings />`:是否显示警告

2. maven 插件[exec-maven-plugin](http://mojo.codehaus.org/exec-maven-plugin/)
  运行任何本地的系统命令。`<id />`:唯一标示命令。`<phase />`:执行时期，prepare-package 预打包时期。`<goals />`:exec 有两种，一种是 exec 另一种是 java。exec 比 java 更强大。`<configuration />`：具体配置。`<executable />`：命令名称。`<workingDirectory />`：执行该命令所处路径。`<arguments />`：可选参数，每个参数都用`<argument>`包起来。`${env}`：引用 pom 定义的变量 env。
这样配置后，我们在云服务器上发布，当执行到这个插件的时候就会根据实际的环境执行相应的命令`cnpm install` `cnpm run build:product`，而系统接收到这个命令后就会去执行对应的 package.json 里面的`build:product`对应的脚本

3. maven 插件[maven-war-plugin](http://maven.apache.org/plugins/maven-war-plugin/)
WAR Plugin 负责收集 web 应用程序的所有工件（项目）依赖项、类和资源，并将它们打包到 web 应用程序存档中。换句话说就是对项目进行动态打包。
maven-war-plugin 会执行两个事情。第一步：复制 webResource 文件`[INFO] Copying webapp webResources [%projectPath%/src/main/filters/webResources/development] to [%projectPath%\target\program-name-0.1.0-SNAPSHOT]`复制 webapp 底下的 resource 文件`[INFO] Copying webapp resources [%projectPath%\src\main\webapp] to [%projectPath%\target\program-name-0.1.0-SNAPSHOT]`。第二步：把`program-name-0.1.0-SNAPSHOT`打成 war 包。
`<webResources />`用来配置`resource`的目录`src/main/filters/webResources/${env}`。`<warSourceIncludes />`用来配置`package阶段`的第一步`coping相关文件`的过滤规则。`<packagingExcludes />`用来配置`package阶段`的第二步`打成war包`的过滤规则。

4. warSourceExcludes 和 packagingExcludes 的区别
warSourceExcludes 是在`compile编译阶段`完成后，`package阶段`的第一步，过滤文件使其不被复制到目录下`program-name-0.1.0-SNAPSHOT`。
packagingExcludes 是在`compile编译阶段`完成后，`package阶段`的第二步，过滤文件使其不被打包到文件内`program-name-0.1.0-SNAPSHOT.war`

5. 也许有人会问，既然这里可以`Excludes`不要的文件，那么第一个问题，js 也是可以放 webapp 下面然后再忽略掉不就可以了？
遗憾的是 maven-war-plugin 是在其他插件（比如 exec-maven-plugin）都执行完之后才执行，因为这个插件的作用是把所有编译后的文件 copy 到即将被打包的`program-name-0.1.0-SNAPSHOT`文件夹下面，所以它必须最后做，也就是说它必须等所有相关的插件都执行完了，才能执行。另外，假如把 js 放到 webapp 下面，即便是使用了`<warSourceIncludes />`来过滤复制的文件，但是因为在复制之前构建索引就要花费了大量的性能，以及复制的过程中消耗了大量的 IO，会占用系统很大的资源，所以有可能会产生内存溢出的问题。

### 进阶问题。Maven 构建的时候能否指定 Coping webResource 和 Coping resource ？ 答案是不能。为什么？
我们前面讲到 webResource 是 webapp 里面的文件，resource 是全局的一些资源文件。maven 会根据自己定义好的标准目录结构，编译`src/main/java`复制`src/main/resources`和`src/main/webapp`到`program-name-0.1.0-SNAPSHOT`然后再打成 war 包。所以，只要是 maven 项目，就必须按照它定义好的目录结构来放文件。

## 总结

前后端一体化自动构建有几个好处

- 发布的时候不用每次 npm run build 只要跟后端的同学一样提交代码，剩下的交给服务器去做。
- config 等全局性质的配置文件不用再通过 url 匹配出对应的值，只要交给云服务器自动把对应环境的配置生效即可。
- 后端的代码前端的同学可以很容易看到，有些问题甚至可以直接改掉，减少不必要的沟通。例如返回的字段错了`item`没加`s`等等显而易见的问题。
- 自从能看到服务端接口的代码后，跟后端的同学讨论问题就更有底气了。
- 一个量化的数据是，原来用 npm run build 因为工程大，项目文件多，每次 run build 都要非常久，然后还要 commit 才能发布。后来经过改造，在`package打包阶段`过滤掉了许多文件大幅减少 IO，同时只打包有用的文件，从整体上提高了发布速度，从原来的将近半个小时缩短为三分钟。
