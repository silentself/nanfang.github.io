---
layout: post
title:  maven-build<1>
excerpt: "maven虽然很流行，但是很多的问题常常无可避免，有句话说的好既然我们无法改变（当前如果是大神也可以重写一下maven的框架），就想办法适应它，发现就记录..."
categories: [java]
tags: [maven]
comments: true
---

#### 1、maven打完包之后显示没有主清单

```consol
no main manifest attribute, in varys-eureka-0.0.1-SNAPSHOT.jar
```

尝试方法1：在MATA-INF/MANIFEST.MF文件中添加如下内容，无果

```txt
Main-Class: com.xxg.Main(主启动类的全类名)
```

尝试方法2：在pom.xml文件中添加如下内容,然后重新打包，成功

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <maimClass>com.guwukeji.varyseureka.VarysEurekaApplication</maimClass>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

分析：将重新打好的包跟之前报错的包对比WEB-INF/MATA-INF/MANIFEST.MF文件

报错的：

```MF
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: tomcat
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_31
```

正常的：

```MF
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: tomcat
Start-Class: com.guwukeji.varyseureka.VarysEurekaApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Version: 2.0.4.RELEASE
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_31
Main-Class: org.springframework.boot.loader.JarLauncher
```

而且最糟糕的是错误的jar里面的文件目录是不全的，比如没有WBE-INF这个文件夹等等

#### 2、偶尔在父工程build（clean）会出现执行失败

报错：在执行clean命令时删除某个子工程的jar/war文件失败

一般而言，重新启动一下IED就好了