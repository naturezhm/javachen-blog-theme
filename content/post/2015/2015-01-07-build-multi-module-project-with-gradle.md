---
layout: post

title: Gradle构建多模块项目
date: 2015-01-07T08:00:00+08:00

categories: [ java ]

tags: [ java,gradle ]

description: 使用 Gradle 构建多模块项目。

published: true

---

废话不多说，直接进入主题。

# 1. 创建项目

首先创建项目，名称为 test：

~~~bash
mkdir test && cd test
gradle init
~~~

这时候的项目结构如下：

~~~
➜  test  tree
.
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
└── settings.gradle

2 directories, 6 files
~~~

然后，创建多个模块，这里以 core 和 web 模块为例，先创建两个目录：

~~~
mkdir -p core/src/main/java
mkdir -p core/src/test/java
mkdir -p web/src/main/java
mkdir -p web/src/test/java
~~~

这时候的项目结构如下：

~~~
➜  test  tree
.
├── build.gradle
├── core
│   └── src
│       ├── main
│       │   └── java
│       └── test
│           └── java
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
└── web
    └── src
        ├── main
        │   └── java
        └── test
            └── java

14 directories, 6 files
~~~

# 2. 修改配置

接下来修改根目录下的 settings.gradle 文件，引入子模块：

~~~
include 'core','web'  
~~~

修改根目录下的 build.gradle：

~~~groovy

// 所有子项目的通用配置
subprojects { 
    apply plugin: 'java'
    apply plugin: 'eclipse'
    apply plugin: 'idea'

    version = '1.0'  

    // JVM 版本号要求
    sourceCompatibility = 1.7
    targetCompatibility = 1.7  

    // java编译的时候缺省状态下会因为中文字符而失败
    [compileJava,compileTestJava,javadoc]*.options*.encoding = 'UTF-8'
    
    //定义版本号
    ext {  
        springVersion = '3.2.11.RELEASE'  
        hibernateVersion='4.3.1.Final'  
    } 

    repositories {
        mavenCentral()
    }

    jar {
        manifest {
            attributes("Implementation-Title": "Gradle")
        }
    }

    configurations {
        // 所有需要忽略的包定义在此
        all*.exclude group: 'commons-httpclient'
        all*.exclude group: 'commons-logging'
        all*.exclude group: 'commons-beanutils', module: 'commons-beanutils'
    }

    dependencies {
        // 通用依赖
        compile(  
                "org.springframework:spring-context:$springVersion",  
                "org.springframework:spring-orm:$springVersion",  
                "org.springframework:spring-tx:$springVersion",  
                "org.springframework.data:spring-data-jpa:1.5.2.RELEASE",  
                "org.hibernate:hibernate-entitymanager:$hibernateVersion",  
                "c3p0:c3p0:0.9.1.2",  
                "mysql:mysql-connector-java:5.1.26",  
                "org.slf4j:slf4j-nop:1.7.6",  
                "commons-fileupload:commons-fileupload:1.3.1",  
                "com.fasterxml.jackson.core:jackson-databind:2.3.1"  
        )

        // 依赖maven中不存在的jar
        ext.jarTree = fileTree(dir: 'libs', include: '**/*.jar')
        ext.rootProjectLibs = new File(rootProject.rootDir, 'libs').getAbsolutePath()
        ext.jarTree += fileTree(dir: rootProjectLibs, include: '**/*.jar')

        compile jarTree

        // 测试依赖
        testCompile(  
                "org.springframework:spring-test:$springVersion",  
                "junit:junit:4.11"  
        ) 
    }

    // 显示当前项目下所有用于 compile 的 jar.
    task listJars(description: 'Display all compile jars.') << {
        configurations.compile.each { File file -> println file.name }
    }
}
~~~

接下来可以修改 core/build.gradle 来定义 core 模块的依赖：

~~~groovy
// jar包的名字
archivesBaseName = 'core'

// 还可以定义其他配置，这里直接继承父模块中的配置
~~~

web 模块需要依赖 core 模块，故定义 web/build.gradle 如下：

~~~groovy
apply plugin:"war"  

dependencies{  
    // 依赖 core 模块
    compile project(":core")  
    compile(  
            "org.springframework:spring-webmvc:$springVersion",  
            "org.apache.taglibs:taglibs-standard-impl:1.2.1"  
    )  
    //系统提供的依赖
    providedCompile(  
            "javax.servlet:javax.servlet-api:3.1.0",  
            "javax.servlet.jsp:jsp-api:2.2.1-b03",  
            "javax.servlet.jsp.jstl:javax.servlet.jsp.jstl-api:1.2.1"  
    )  
}  

task jarWithoutResources(type: Jar) {  
    baseName project.name  
    from("$buildDir/classes/main")  
}  

war{  
    dependsOn jarWithoutResources  
    from("$projectDir/src/main/resources") {  
        include "*.properties"  
        into("WEB-INF/classes")  
    }  
    classpath=classpath - sourceSets.main.output  
    classpath fileTree(dir:libsDir, include:"${project.name}-${version}.jar")  
}  
task('jarPath')<<{  
    configurations.runtime.resolve().each {  
        print it.toString()+";"  
    }  
    println();  
}  
~~~

# 3. 编译项目

查看所有 jar：

~~~bash
$ gradle listJars
~~~

查看各个模块的依赖：

~~~bash
$ gradle :core:dependencies
$ gradle :web:dependencies
~~~

编译所有模块：

~~~bash
$ gradle build
~~~

对比一下，这时候的目录如下：

~~~
➜  test  tree
.
├── build.gradle
├── core
│   ├── build
│   │   ├── libs
│   │   │   └── core-1.0.jar
│   │   └── tmp
│   │       └── jar
│   │           └── MANIFEST.MF
│   ├── build.gradle
│   └── src
│       ├── main
│       │   └── java
│       └── test
│           └── java
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
└── web
    ├── build
    │   ├── libs
    │   │   ├── web-1.0.jar
    │   │   └── web-1.0.war
    │   └── tmp
    │       ├── jarWithoutResources
    │       │   └── MANIFEST.MF
    │       └── war
    │           └── MANIFEST.MF
    ├── build.gradle
    └── src
        ├── main
        │   └── java
        └── test
            └── java

23 directories, 14 files
~~~

这样，core和web模块都是gradle项目了，你也可以单独编译某一个模块，例如，编译core模块：

~~~bash
$ cd core
$ rm -rf build
$ gradle build
$ tree
.
├── build
│   ├── libs
│   │   └── core-1.0.jar
│   └── tmp
│       └── jar
│           └── MANIFEST.MF
├── build.gradle
└── src
    ├── main
    │   └── java
    └── test
        └── java

9 directories, 3 files
~~~

# 4. 一些小技巧

## 1. 善用 gradle dependencies

~~~bash
gradle dependencies > depend.log
~~~

## 2. java 编译时候报编码错误

~~~groovy
[compileJava,compileTestJava,javadoc]*.options*.encoding = 'UTF-8'
~~~

## 3. 忽略掉 .gradle 目录

修改 .gitignore 忽略该目录：

~~~
*.sw?
.#*
*#
*~
.classpath
.project
.settings
bin
build
target
dependency-reduced-pom.xml
*.sublime-*
/scratch
.gradle
README.html
.idea
*.iml
~~~

## 4. Maven 库中没有的 jar 该怎么管理

在顶级目录增加一个 libs 文件夹，这个文件夹里面的 jar 是对所有项目都起作用的。

如果是某个项目自用的，则可以在该项目的 source 下面创建个 libs，具体实现是在顶级目录下的 build.gradle 中：

~~~groovy
ext.jarTree = fileTree(dir: 'libs', include: '**/*.jar')
ext.rootProjectLibs = new File(rootProject.rootDir, 'libs').getAbsolutePath()
ext.jarTree += fileTree(dir: rootProjectLibs, include: '**/*.jar')

compile jarTree
~~~

## 5. jar 包定义外移

暂时还没有这样的需求，详细说明请参考 [jar 包定义外移](https://github.com/someok/gradle-multi-project-example/blob/master/readme.md#jar-%E5%8C%85%E5%AE%9A%E4%B9%89%E5%A4%96%E7%A7%BB)

## 6. 如何指定 build 输出目录和版本号

~~~
buildDir = "target"
version = '1.0'
~~~

## 7. 在执行 Gradle 命令时如何指定参数

~~~
gradle task -P profile=development
~~~

## 8. Gradle 和 idea 集成时如何不自动下载依赖源码和javadoc

~~~groovy
idea {
    module {
        downloadJavadoc = false
        downloadSources = false
    }
}
~~~

# 5. 参考文章

- [gradle多模块开发](http://blog.csdn.net/xiejx618/article/details/38469865)
- [Gradle 多项目管理示例](https://github.com/someok/gradle-multi-project-example/blob/master/readme.md)
- [构建工具之 - Gradle一般使用常见问答](http://www.cnblogs.com/xguo/p/3175377.html)
