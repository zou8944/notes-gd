# Gradle学习笔记

## 为什么使用Gradle

### Maven的特点

Maven将整体的构建过程抽象成固定的分离的生命周期，如clean、compile等，再通过一系列官方提供的已有插件在不同的声明周期完成不同的任务，而用户使用Maven的核心——pom.xml仅以配置文件的形式呈现。通过配置文件，我们声明依赖、声明插件，最大程度地将配置中的重复部分抽离出来，极大地实现了复用。

使用中，开发者只需要在约定的位置编写代码和放置配置文件，再在配置文件中声明依赖项和插件，剩下的部分Maven就能很好滴帮我们完成。

### Gradle比Maven好在哪里

Maven的问题有两点

- 最大的问题，就是仅对用户暴露配置文件，如果需要执行较为复杂的自定义操作，比如动态读取配置文件，就需要我们自定义自己的插件，这显得门槛过高且复杂。
- Maven的约定是一个项目只包含一个Java源代码路径，只产生一个JAR文件。
- Maven构建生命周期是线性的，且必然会从头开始，不方便自定义。

Gradle相比而言的优点

- 使用Groovy来自定义我们的构建逻辑，对于一般的自定义需求可以直接在配置文件中使用Groovy完成

- 能够集成Ant和Maven，项目可以很容易地从他们迁到Gradle

  Gradle生成的构建可以发布到Maven仓库供Maven使用。

- 允许一个项目包含多个Java源代码路径，产生多个JAR文件

- Gradle的构建生命周期较为灵活，方便自定义。

### Gradle有什么缺点吗

- 太灵活，Maven已经可以应对绝大多数情况
- 只是对Ant和Maven更加优秀的实现，没有引入新的概念

## Gradle简笔记

- Maven的pom.xml对应Gradle的build.gradle

### Java Plugin

- Gradle也提供类似Maven的约定优于配置，这是通过Java Plugin实现的，gradle也推荐这种方式，它的文件布局方式和Maven一样

  - src/main/java
  - src/main/resources
  - src/test/java
  - src/test/resources

  唯一的区别是在Gradle中要修改这个默认配置更加容易

  ```groovy
  apply plugin: 'java'
  
  sourceSets {
      main {
          java {
              srcDir 'src/java'
          }
          resources {
              srcDir 'src/resources'
          }
      }
  }
  ```

  此外，Java Plugin也定义了构建的生命周期，包括编译主代码、处理资源、编译测试代码、执行测试、上传归档等等任务

### 最简单的构建

```groovy
apply plugin: 'java'
```

这一行代码就足以构建一个常规的Java项目

- gradle build —— 执行build任务，这是java插件加进来的

  ```groovy
  floyd@floyd-ThinkPad-T490:~/PersonalCode/Asynchronous$ gradle build
  :compileJava NO-SOURCE
  :processResources NO-SOURCE
  :classes UP-TO-DATE
  :jar
  :assemble
  :compileTestJava NO-SOURCE
  :processTestResources NO-SOURCE
  :testClasses UP-TO-DATE
  :test NO-SOURCE
  :check UP-TO-DATE
  :build
  
  BUILD SUCCESSFUL in 1s
  1 actionable task: 1 executed
  ```

  gradle会自动检查哪些任务没有发生变化，从而跳过执行，加快构建速度。

### 自定义构建

Java插件是一个非常固执的框架，对于项目很多的方面它都假定有默认值，比如项目布局，如果你看待世界的方法是不一样的，Gradle给你提供了一个自定义约定的选项。

https://docs.gradle.org/current/userguide/java_plugin.html

### 命令行

- gradle tasks —— 显示当前构建文件的所有任务
- gradle taskName —— 指定指定名称的任务
- gradle taskName -x excludeTaskName —— 运行任务时，排除某个任务

### gradle包装器

包装器是Gradle的一个核心特性，它允许你的机器不需要安装运行时就能运行Gradle脚本，而且她还能确保build脚本运行在指定版本的Gradle。它会从中央仓库中自动下载Gradle运行时，解压到你的文件系统，然后用来build。终极目标就是创建可靠的、可复用的、与操作系统、系统配置或Gradle版本无关的构建。

- 生成

  在项目下执行gradle wrapper就能生成包装器，默认生成你正在使用的版本，也可以自定义版本。

  ![image20200531170716242](./Gradle学习笔记/image-20200531170716242.png?stamp=0)

  ![image20200531170903347](./Gradle学习笔记/image-20200531170903347.png?stamp=0)

- 使用

  上面生成两个脚本，gradlew是给linux系统使用的，gradlew.bat是给window系统使用的

  ![image20200531171030256](./Gradle学习笔记/image-20200531171030256.png?stamp=0)

### 构建块

每个Gradle构建都包括三个基本的构建块：项目(projects)、任务(tasks)和属性(properties)，每个构建至少包括一个项目，项目包括一个或者多个任务，项目和任务都有很多个属性来控制构建过程。

#### 项目

表示你想构建的一个组件(如一个jar包)，或者你想完成的一个目标(如打包APP)。

当开始构建过程后，Gradle基于你的配置实例化org.gradle.api.Project这个类以及让这个项目通过project变量来隐式的获得。

![img](https://lippiouyang.gitbooks.io/gradle-in-action-cn/content/images/dag24.png)

#### 任务

你应该了解两个概念：任务动作(actions)和任务依赖，一个动作就是任务执行的时候一个原子的工作，这可以简单到打印hello world,也可以复杂到编译源代码。

Gradle API中任务的表示：org.gradle.api.Task 接口。

![img](https://lippiouyang.gitbooks.io/gradle-in-action-cn/content/images/dag25.png)

**动作**

动作就是在你的任务中放置构建逻辑的地方，Task接口提供了两个方法来声明任务的动作： doFirst和doLast，当任务执行的时候，定义在闭包里的动作逻辑就按顺序执行。

对于每个任务可以有多个动作，实际上，当任务创建的时候你可以添加任意多个动作，每一个任务都有一个动作清单。

**任务依赖**

dependsOn方法用来声明一个任务依赖于一个或者多个任务

```groovy
task first << { println "first" }
task second << { println "second" }

//声明多个依赖
task printVersion(dependsOn: [second, first]) << {
    logger.quiet "Version: $version"
}

task third << { println "third" }
//通过任务名称来声明依赖
third.dependsOn('printVersion')
```

Gradle并**不保证依赖的任务能够按顺序执行**，dependsOn方法只是定义这些任务应该在这个任务之前执行，但是这些依赖的任务具体怎么执行它并不关心

**终结者任务**

可以不进行依赖指定终结者任务

```groovy
task first << { println "first" }
task second << { println "second" }
//声明first结束后执行second任务
first.finalizedBy second
```

**自定义任务**

Gradle会给每一个任务创建一个DefaultTask类型的实例，当你要创建一个自定义的任务时，你需要创建一个继承自DefaultTask的类

```groovy
class ReleaseVersionTask extends DefaultTask {
    //通过注解声明任务的输入和输出    
    @Input Boolean release
    @OutputFile File destFile

    ReleaseVersionTask() {
        //在构造器里设置任务的分组和描述
        group = 'versioning'
        description = 'Makes project a release version.'
    }
    //通过注解声明要执行的任务
    @TaskAction
    void start() {
        project.version.release = true
        ant.propertyfile(file: destFile) {
            entry(key: 'release', type: 'string', operation: '=', value: 'true')
            }
    }
}
```

#### 属性

每个Project和Task实例都提供了setter和getter方法来访问属性，属性可以是任务的描述或者项目的版本号

**外部属性**

外部属性一般存储在键值对中，要添加一个属性，你需要使用ext命名空间，看一个例子：

```groovy
project.ext.myProp = 'myValue'
ext {
   someOtherProp = 123
}

//Using ext namespace to access extra property is optional
assert myProp == 'myValue'
println project.someOtherProp
ext.someOtherProp = 567    
```

外部属性可以定义在一个属性文件中： 通过在/.gradle路径或者**项目根目录下的 gradle.properties**文件来定义属性可以直接注入到你的项目中，他们可以通过 project实例来访问

### 构建生命周期

作为一个构建脚本的开发者，你不应该局限于编写任务动作或者配置逻辑，有时候你想在指定的生命周期事件发生的时候执行一段代码。生命周期事件可以在指定的生命周期之前、之中或者之后发生，在执行阶段之后发生的生命周期事件就该是构建的完成了。

![img](https://lippiouyang.gitbooks.io/gradle-in-action-cn/content/images/dag27.png)

在配置阶段，Gradle决定在任务在执行阶段的执行顺序，依赖关系的内部结构是通过直接的无环图(DAG)来表示的，图中的每一个任务称为一个节点，每一个节点通过边来连接，你很有可能通过dependsOn或者隐式的依赖推导来创建依赖关系。记住DAG图从来不会有环，就是说一个已经执行的任务不会再次执行，下面这幅图将要的展示了这个过程：

![img](https://lippiouyang.gitbooks.io/gradle-in-action-cn/content/images/dag28.png)

```groovy
gradle.taskGraph.whenReady { TaskExecutionGraph taskGraph ->
   //检查任务图是否包括release任务
    if(taskGraph.hasTask(release)) {

        if(!version.release) {

            version.release = true

            ant.propertyfile(file: versionFile) {
                entry(key: 'release', type: 'string', operation: '=',
                value: 'true')
            }
        }
    }
}
```

你也可以实现一个监听器来实现同样的效果

### 依赖管理

在Java领域里支持声明的自动依赖管理的有两个项目：Apache Ivy(Ant项目用的比较多的依赖管理器)和Maven(在构建框架中包含一个依赖管理器)，我不再详细介绍这两个的细节而是解释自动依赖管理的概念和机制。

Ivy和Maven是通过XML描述文件来表达依赖配置，配置包含两部分：依赖的标识加版本号和中央仓库的位置(可以是一个HTTP链接)，依赖管理器根据这个信息自动定位到需要下载的仓库然后下载到你的机器中。库可以定义传递依赖，依赖管理器足够聪明分析这个信息然后解析下载传递依赖。如果出现了依赖冲突比如上面的Hibernate core的例子，依赖管理器会试着解决。库一旦被下载就会存储在本地的缓存中，构建系统先检查本地缓存中是否存在需要的库然后再从远程仓库中下载。

Gradle通过DSL来描述依赖配置，实现了上面描述的架构。

DSL配置block dependencies用来给配置添加一个或多个依赖，你的项目不仅可以添加外部依赖，下面这张表显示了Gradle支持的各种不同类型的依赖。

![img](https://lippiouyang.gitbooks.io/gradle-in-action-cn/content/images/5-3.png)

**依赖属性**

当依赖管理器从仓库中查找依赖时，需要通过属性的结合来定位，最少需要提供一个name。

- group： 这个属性用来标识一个组织、公司或者项目，可以用点号分隔，Hibernate的group是org.hibernate。
- name： name属性唯一的描述了这个依赖，hibernate的核心库名称是hibernate-core。
- version： 一个库可以有很多个版本，通常会包含一个主版本号和次版本号，比如Hibernate核心库3.6.3-Final。
- classifier： 有时候需要另外一个属性来进一步的说明，比如说明运行时的环境，Hibernate核心库没有提供classifier。

你可以使用下面的语法在项目中声明依赖：

```
dependencies {
    configurationName dependencyNotation1,     dependencyNotation2, ...
}
```

你先声明你要给哪个配置添加依赖，然后添加依赖列表，你可以用map的形式来注明，你也可以直接用冒号来分隔属性，比如这样的：

![img](https://lippiouyang.gitbooks.io/gradle-in-action-cn/content/images/5-5.png)

```groovy
dependencies {
    //使用映射声明依赖
    compile group: cargoGroup, name: 'cargo-core-uberjar', version: cargoVersion
    //用快捷方式来声明，引用了前面定义的外部属性
    cargo "$cargoGroup:cargo-ant:$cargoVersion"
}
```

**排除传递依赖**

Gradle允许你完全控制传递依赖，你可以选择排除全部的传递依赖也可以排除指定的依赖，假设你不想使用UberJar传递的xml-api的版本而想声明一个不同版本，你可以使用exclude方法来排除它：

```
dependencies {
    cargo('org.codehaus.cargo:cargo-ant:1.3.1') {
        exclude group: 'xml-apis', module: 'xml-apis'
    }
    cargo 'xml-apis:xml-apis:2.0.2'
}
```

exclude属性值和正常的依赖声明不太一样，你只需要声明group和(或)module，Gradle不允许你只排除指定版本的依赖。

有时候仓库中找不到项目依赖的传递依赖，这会导致构建失败，Gradle允许你使用transitive属性来排除所有的传递依赖：

```groovy
dependencies {
    cargo('org.codehaus.cargo:cargo-ant:1.3.1') {
    transitive = false
    }
    // 选择性的声明一些需要的库
}
```

**动态版本声明**

如果你想使用一个依赖的最新版本，你可以使用latest.integration，比如声明 Cargo Ant tasks的最新版本，你可以这样写 `org.codehaus .cargo:cargo-ant:latest-integration`，你也可以用一个+号来动态的声明：

```groovy
dependencies {
    //依赖最新的1.x版本
    cargo 'org.codehaus.cargo:cargo-ant:1.+'
}
```

**仓库**

Gradle支持下面三种不同类型的仓库：

![img](https://lippiouyang.gitbooks.io/gradle-in-action-cn/content/images/5-8.png)

# 第一次搭建gradle kotlin环境

搭建步骤

1. 使用IDEA选择gradle，kotlin生成新的项目。
2. 等待下载gradle完成

问题：

1. 出现找不到org.jetbrains.kotlin.jvm插件的问题

   在settings.gradle中增加插件管理依赖

   ```gradle
   // 放在文件最开头
   pluginManagement {
       repositories {
           mavenCentral()
           gradlePluginPortal()
       }
   }
   ```

2. 没有基本的项目目录

   自己创建folder，IDEA也会提示创建如下几个文件夹的快捷方式

   - src/main/kotlin
   - src/main/resources
   - src/test/kotlin
   - src/test/resources

# 问题

Maven插件和Gradle插件可以互用吗？