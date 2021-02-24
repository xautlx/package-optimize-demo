### Level 0：常规的Fat Jar构建

参考项目目录：package-optimize-level0

**主要配置：**

~~~xml
<build>
    <finalName>${project.artifactId}</finalName>
    <!--
    特别注意：
    项目仅仅是为了演示配置方便，直接在parent的build部分做了插件配置和运行定义。
    但是实际项目中需要把这些定义只放到spring boot模块项目（可优化使用pluginManagement形式），避免干扰其他util、common等模块项目
    -->
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
~~~

**配置输出：**

~~~shell script
cd package-optimize-level0
mvn clean install

ls -lh package-optimize-app1/target/package-optimize-app1.jar
-rw-r--r--  1 lixia  wheel    16M Feb 24 21:06 package-optimize-app1/target/package-optimize-app1.jar

java -jar package-optimize-app1/target/package-optimize-app1.jar
~~~

**重点说明：**

* （当前演示应用仅依赖了spring-boot-starter-web极少组件，所有构建输出只有十来MB）实际情况单一构建根据项目依赖组件量输出jar一般在几十MB到一两百MB。
* 假如有十来个微服务需要部署，那就意味着需要传输一两个GB的文件，耗时可想而知。

### Level 1：常见的依赖jar分离构建方式

参考项目目录：package-optimize-level1

**关解决问题：**

* 降低单个微服务jar的文件大小，以便部署过程秒传文件。

**主要配置：**

~~~xml
<build>
    <finalName>${project.artifactId}</finalName>
    <!--
    特别注意：
    项目仅仅是为了演示配置方便，直接在parent的build部分做了插件配置和运行定义。
    但是实际项目中需要把这些定义只放到spring boot模块项目（可优化使用pluginManagement形式），避免干扰其他util、common等模块项目
    -->
    <plugins>
        <!-- 拷贝项目所有依赖jar文件到构建lib目录下 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <executions>
                <execution>
                    <id>copy-dependencies</id>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/lib</outputDirectory>
                        <excludeTransitive>false</excludeTransitive>
                        <stripVersion>false</stripVersion>
                        <silent>true</silent>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        <!-- Spring Boot模块jar构建 -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <includes>
                    <!-- 不存在的include引用，相当于排除所有maven依赖jar，没有任何三方jar文件打入输出jar -->
                    <include>
                        <groupId>null</groupId>
                        <artifactId>null</artifactId>
                    </include>
                </includes>
                <layout>ZIP</layout>
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
~~~

**配置输出：**

~~~shell script
cd package-optimize-level1
mvn clean install

ls -lh package-optimize-app1/target/package-optimize-app1.jar
-rw-r--r--  1 lixia  wheel   149K Feb 24 20:56 package-optimize-app1/target/package-optimize-app1.jar

java -jar -Djava.ext.dirs=lib package-optimize-app1/target/package-optimize-app1.jar
~~~


**实现效果：**

* 单一构建根据项目依赖组件量输出jar一般仅有一两百KB，基本可以做到秒传。
* 这种方式有个明显问题：假如有十来个微服务，每个服务一个jar和一个lib目录文件，首次部署也差不多需要传输一两个GB文件。

### Level 2：合并所有模块依赖jar到同一个lib目录

参考项目目录：package-optimize-level2

**解决问题：**

* 合并所有模块依赖jar到同一个lib目录，一般由于各模块项目依赖jar重叠程度很高，合并所有服务部署文件总计大小基本也就两三百MB
* 但是如果采用-Djava.ext.dirs=lib加载所有jar到每个JVM，一来每个JVM都完整加载了所有jar耗费资源，二来各微服务组件版本不同会出现版本冲突问题

**主要配置：**

~~~xml
<build>
    <finalName>${project.artifactId}</finalName>
    <!--
    特别注意：
    项目仅仅是为了演示配置方便，直接在parent的build部分做了插件配置和运行定义。
    但是实际项目中需要把这些定义只放到spring boot模块项目（可优化使用pluginManagement形式），避免干扰其他util、common等模块项目
    -->
    <plugins>
        <!-- 拷贝项目所有依赖jar文件到构建lib目录下 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <executions>
                <execution>
                    <id>copy-dependencies</id>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/lib</outputDirectory>
                        <excludeTransitive>false</excludeTransitive>
                        <stripVersion>false</stripVersion>
                        <silent>true</silent>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        <!-- Spring Boot模块jar构建 -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <includes>
                    <!-- 不存在的include引用，相当于排除所有maven依赖jar，没有任何三方jar文件打入输出jar -->
                    <include>
                        <groupId>null</groupId>
                        <artifactId>null</artifactId>
                    </include>
                </includes>
                <layout>ZIP</layout>
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
~~~

**配置输出：**

~~~shell script
cd package-optimize-level1
mvn clean install

ls -lh package-optimize-app1/target/package-optimize-app1.jar
-rw-r--r--  1 lixia  wheel   149K Feb 24 20:56 package-optimize-app1/target/package-optimize-app1.jar

java -jar -Djava.ext.dirs=lib package-optimize-app1/target/package-optimize-app1.jar
~~~


**实现效果：**

* 单一构建根据项目依赖组件量输出jar一般仅有一两百KB，基本可以做到秒传。
* 这种方式有个明显问题：假如有十来个微服务，每个服务一个jar和一个lib目录文件，首次部署也差不多需要传输一两个GB文件。

