---
share: true
---

> https://juejin.cn/post/7248207744087277605


## 创建 Task
### register
创建 Task 需要使用 TaskContainer 的 register 方法。

register的几种方式：

```
1. register(String name, Action<? super Task> configurationAction)
2. register(String name, Class type, Action<? super T> configurationAction)
3. register(String name, Class type)
4. register(String name, Class type, Object... constructorArgs)
5. register(String name)
```

  
比较常用的是1和2。
- configurationAction指的是Action，也就是该Task的操作，会在编译时执行；
- type 类型指的是 Task 类型，可以是自定义类型，也可以指定自带的 Copy、Delete、Zip、Jar 等类型

```
tasks.register("yechaoa") {
    println("Task Name = " + it.name)
}
```

上面task我们是通过TaskContainer(tasks)创建的，在Project对象中也提供了创建Task的方法，写法上有一点差异：

```
task("yechaoa") {
    println "aaa"
}
```


### create
创建 Task 除了上面示例中的 `register` 方法，还有 `create` 方法，那它们有什么区别呢?

- 通过register创建时，只有在这个task被需要时，才会创建和配置；
- 通过create创建时，则会立即创建与配置该Task，并添加到TaskContainer中；


register 是按需创建 task 的方式，这样 gradle 执行的性能更好
需要注意的是，register属于懒加载，嵌套创建的Task在配置阶段无法被初始化，所以并不会被执行到。



## 执行 Task

```
./gradlew taskname
```

## Task执行结果
### EXCUTED
表示Task执行，常见
### UP-TO-DATE
```
> Task :app:preBuild UP-TO-DATE
```
它表示 Task 的输出没有改变
分几种情况：
1. 输入和输出都没有改变；
2. 输出没有改变；
3. Task没有操作，有依赖，但依赖的内容是最新的，或者跳过了，或者复用了；
4. Task 没有操作，也没有依赖；

### FOME-CACHE
从缓存中复用上一次的执行结果

### SKIPPED
跳过。比如被排除
```
$ gradle dist --exclude-task yechaoa
```

### NO-SOURCE
Task不需要执行。有输入和输出，但没有来源。


## Task 的 Action

### 自定义Task

```
class YechaoaTask extends DefaultTask {

    @Internal
    def taskName = "default"

    @TaskAction
    def MyAction1() {
        println("$taskName -- MyAction1")
    }

    @TaskAction
    def MyAction2() {
        println("$taskName -- MyAction2")
    }
}

```
- 自定义一个类，继承自 `DefaultTask`；
- Action的方法需要添加`@TaskAction`注解；
- 对外暴露的参数需要使用`@Internal`注解；

```
tasks.register("yechaoa", YechaoaTask) {
    taskName = "我是传入的Task Name "
}
```

```
> Task :app:yechaoa
我是传入的Task Name  -- MyAction2
我是传入的Task Name  -- MyAction1
```


### doFirst、doLast

```
tasks.register("yechaoa") {
    it.doFirst {
        println("${it.name} = doFirst 111")
    }
    it.doFirst {
        println("${it.name} = doFirst 222")
    }

    println("Task Name = " + it.name)

    it.doLast {
        println("${it.name} = doLast 111")
    }
    it.doLast {
        println("${it.name} = doLast 222")
    }
}
```


```
Task Name = yechaoa

> Task :app:yechaoa
yechaoa = doFirst 222
yechaoa = doFirst 111
yechaoa = doLast 111
yechaoa = doLast 222
```

Task Name 的输出是在 Gradle 生命周期的配置阶段，因为它就在闭包下面，不在任何 Action 里，没有执行时机，配置阶段解析到这个 Task 就会执行 println。

其他输出都是在Task :app:yechaoa下，因为有明确的Action执行时机。

### Action执行顺序
只有 doLast 是正序的，其余都是倒序执行

![](assets/Pasted%20image%2020240617150002.png)



## Task属性

```
String TASK_NAME = "name";
String TASK_DESCRIPTION = "description";
String TASK_GROUP = "group";
String TASK_TYPE = "type";
String TASK_DEPENDS_ON = "dependsOn";
String TASK_OVERWRITE = "overwrite";
String TASK_ACTION = "action";

tasks.register("yechaoa") {
	it.configure {
		group = "help"
		name = ""
		dependsOn = ""
	}
}
```

### dependsOn

```
tasks.register("yechaoa111") {
    it.configure {
        dependsOn(provider {
            tasks.findAll {
                it.name.contains("yechaoa222")
            }
        })
    }

    it.doLast {
        println("${it.name}")
    }
}

tasks.register("yechaoa222") {
    it.doLast {
        println("${it.name}")
    }
}

```

```
./gradlew yechaoa111
====
> Task :app:yechaoa222
yechaoa222

> Task :app:yechaoa111
yechaoa111
```

简化写法
```
def yechaoa111 = tasks.register("yechaoa111") {
    it.doLast {
        println("${it.name}")
    }
}

def yechaoa222 = tasks.register("yechaoa222") {
    it.doLast {
        println("${it.name}")
    }
}

yechaoa111.configure {
    dependsOn yechaoa222
}
```

dependsOn 依赖的 Task 可以是名称也可以是 path、type类型
```
dependsOn tasks.withType(Copy)

dependsOn "project-lib:yechaoa"

```

### finalizedBy
为Task添加指定的终结器任务。也就是指定下一个执行的Task，dependsOn指定的是上一个。
```
task taskY {
    finalizedBy "taskX"
}
```
这里表示 taskY 执行之后执行 taskX。
如果finalizedBy换成dependsOn，则表示taskY执行前要先执行taskX。

### mustRunAfter
使用 mustRunAfter 定义两个任务执行顺序时，需要两个任务都执行
```
./gradlew yechaoa111 yechaoa222
```

### shouldRunAfter
mustRunAfter 是「必须运行」，shouldRunAfter 是「应该运行」。

如taskB.mustRunAfter(taskA)，当taskA和taskB同时运行时，则taskB必须始终在taskA之后运行。

shouldRunAfter规则类似，但不太一样，因为它在两种情况下会被忽略。首先，如果使用该规则会引入一个排序周期；其次，当使用并行执行时，除了“应该运行”任务外，任务的所有依赖项都已满足，那么无论其“应该运行”依赖项是否已运行，都将运行此任务。

## 跳过Task
条件跳过、异常跳过、禁用跳过、超时跳过

### 条件跳过
Gradle提供了`onlyIf(Closure onlyIfClosure)`方法，只有闭包的结果返回True时，才执行Task。
```
tasks.register("skipTask") { taskObj ->
    taskObj.configure {
        onlyIf {
            def provider = providers.gradleProperty("yechaoa")
            provider.present
        }
    }

    taskObj.doLast {
        println("${it.name} is Executed")
    }
}
```

```
./gradlew skipTask -Pyechaoa
====
> Task :app:skipTask
skipTask is Executed
```


### 异常跳过
如果 onlyIf 不满足需求，也可以使用 `StopExecutionException` 来跳过。

StopExecutionException属于异常，当抛出异常的时候，会跳过当前Action及后续Action，即跳过当前Task执行下一个Task。
### 禁用跳过
每个Task 都有一个`enabled`开关，true开启，false禁用，禁用之后任何操作都不会被执行。

### 超时跳过

Task 提供了 `timeout` 属性用于限制执行时间。
如果Task的运行时间超过指定的时间，则执行该任务的线程将被中断。


## Task增量构建

`增量构建` 是当 Task 的输入和输出没有变化时，跳过 action 的执行，当 Task 输入或输出发生变化时，在 action 中只对发生变化的输入或输出进行处理，这样就可以避免一个没有变化的 Task 被反复构建，当 Task 发生变化时也只处理变化部分，这样可以提高 Gradle 的构建效率，缩短构建时间。


### 增量构建的两种形式

- 第一种，Task完全可以复用，输入和输出都没有任何变化，即UP-TO-DATE；
- 第二种，有部分变化，只需要针对变化的部分进行操作；


```groovy
class CopyTask extends DefaultTask {

    // 指定输入
    @InputFiles
    FileCollection from

    // 指定输出
    @OutputDirectory
    Directory to

    // task action 执行
    @TaskAction
    def execute() {
        File file = from.getSingleFile()
        if (file.isDirectory()) {
            from.getAsFileTree().each {
                copyFileToDir(it, to)
            }
        } else {
            copyFileToDir(from, to)
        }
    }

    /**
     * 复制文件到文件夹
     * @param src 要复制的文件
     * @param dir 接收的文件夹
     * @return
     */
    private static def copyFileToDir(File src, Directory dir) {
        File dest = new File("${dir.getAsFile().path}/${src.name}")

        if (!dest.exists()) {
            dest.createNewFile()
        }

        dest.withOutputStream {
            it.write(new FileInputStream(src).getBytes())
        }
    }

}

```

在编写 Task 的时候，我们需要使用注解来声明输入和输出。`@InputXXX` 表示输入，`@OutputXXX` 表示输出。

但这样的细粒度还不够，可以通过在 Action 增加参数来细分

给 Action 方法增加一个 `InputChanges` 参数，带 InputChanges 类型参数的 Action 方法表示这是一个增量任务操作方法，该参数告诉 Gradle，该 Action 方法仅需要处理更改的输入，此外，Task 还需要通过使用 `@Incremental` 或 `@SkipWhenEmpty` 来指定至少一个增量文件输入属性。

  
```groovy
class CopyTask extends DefaultTask {

    // 指定增量输入属性
    @Incremental
    // 指定输入
    @InputFiles
    FileCollection from

    // 指定输出
    @OutputDirectory
    Directory to

    // task action 执行
    @TaskAction
    void execute(InputChanges inputChanges) {

        boolean incremental = inputChanges.incremental
        println("isIncremental = $incremental")

        inputChanges.getFileChanges(from).each {
            if (it.fileType != FileType.DIRECTORY) {
                ChangeType changeType = it.changeType
                String fileName = it.file.name
                println("ChangeType = $changeType , ChangeFile = $fileName")

                if (changeType != ChangeType.REMOVED) {
                    copyFileToDir(it.file, to)
                }
            }
        }
        
    }

    /**
     * 复制文件到文件夹
     * @param src 要复制的文件
     * @param dir 接收的文件夹
     * @return
     */
    static def copyFileToDir(File file, Directory dir) {
        File dest = new File("${dir.getAsFile().path}/${file.name}")

        if (!dest.exists()) {
            dest.createNewFile()
        }

        dest.withOutputStream {
            it.write(new FileInputStream(file).getBytes())
        }
    }

}

```

1. 给 from 属性增加 `@Incremental` 注解，表示增量输入属性；
2. 重写了action方法execute()，增加了`InputChanges`参数，支持增量复制文件，然后根据文件的`ChangeType`做校验，只复制新增或修改的文件。


### 无法增量的情况
有以下几种情况会全量构建：

- 该 Task 是第一次执行；
- 该Task只有输入没有输出；
- 该Task的upToDateWhen条件返回了false；
- 自上次构建以来，该Task的某个输出文件已更改；
- 自上次构建以来，该Task的某个属性输入发生了变化，例如一些基本类型的属性；
- 自上次构建以来，该Task的某个非增量文件输入发生了变化，非增量文件输入是指没有使用@Incremental或@SkipWhenEmpty注解的文件输入

### 增量构建原理
在首次执行 Task 之前，Gradle 会获取输入的指纹，此指纹包含输入文件的路径和每个文件内容的散列。然后执行 Task，如果 Task 成功完成，Gradle 会获取输出的指纹，此指纹包含一组输出文件和每个文件内容的散列，Gradle 会在下次执行 Task 时保留两个指纹。

后续每次在执行Task之前，Gradle都会对输入和输出进行新的指纹识别，如果新指纹与之前的指纹相同，Gradle假设输出是最新的，并跳过Task，如果它们不一样，Gradle会执行Task。Gradle会在下次执行Task时保留两个指纹。

如果文件的统计信息（即lastModified和size）没有改变，Gradle将重复使用上次运行的文件指纹，即当文件的统计信息没有变化时，Gradle不会检测到更改。

Gradle还将Task的代码视为任务输入的一部分，当Task、Action或其依赖项在执行之间发生变化时，Gradle认为该Task是过时的。

Gradle了解文件属性（例如持有Java类路径的属性）是否对顺序敏感，当比较此类属性的指纹时，即使文件顺序发生变化，也会导致Task过时。

请注意，如果Task指定了输出目录，则自上次执行以来添加到该目录的任何文件都会被忽略，并且不会导致Task过时，如此不相关的Task可能会共享一个输出目录，而不会相互干扰，如果出于某种原因这不是你想要的行为，请考虑使用TaskOutputs.upToDateWhen（groovy.lang.Closure）。

另请注意，更改不可用文件的可用性（例如，将损坏的符号链接的目标修改为有效文件，反之亦然），将通过最新检查进行检测和处理。

Task的输入还用于计算启用时用于加载Task输出的构建缓存密钥。

## 查找Task

查找 Task，主要涉及到 TaskContainer 对象，顾名思义，Task 容器的管理类，它提供了两个方法：

- findByPath(String path)，参数可空
- getByPath(String path)，参数可空，找不到Task会抛异常UnknownTaskException

同时，TaskContainer继承自`TaskCollection`和`NamedDomainObjectCollection`，又增加了两个方法可以使用：

- findByName
- getByName

参数定义与xxxByPath方法一样。

### findByName
```groovy
def aaa = tasks.findByName("yechaoa").doFirst {
    println("yechaoa excuted doFirst by findByName")
}
```
找到一个名为「yechaoa」的Task，并增加一个doFirst Action，然后在doFirst中打印日志。
这时候执行aaa是不会触发yechaoa Task Action的执行，因为并没有依赖关系，所以得执行yechaoa Task。
```
 ./gradlew yechaoa
 ===
 > Task :app:yechaoa
yechaoa excuted doFirst by findByName

```

还有如 named、findByPath 等方法

## Task Tree
可以通过 `./gradlew tasks` 来查看所有的 Task，但却看不到 Task 的依赖关系。

要查看Task的依赖关系，我们可以使用[task-tree](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fdorongold%2Fgradle-task-tree "https://github.com/dorongold/gradle-task-tree")插件
```
plugins {
    id "com.dorongold.task-tree" version "2.1.1"
}
```

```
./gradlew <task 1>...<task N> taskTree
===

:app:build
+--- :app:assemble
|    +--- :app:assembleDebug
|    |    +--- :app:mergeDebugNativeDebugMetadata
|    |    |    --- :app:preDebugBuild
|    |    |         --- :app:preBuild
|    |    --- :app:packageDebug
|    |         +--- :app:compileDebugJavaWithJavac
|    |         |    +--- :app:compileDebugAidl
|    |         |    |    --- :app:preDebugBuild *
|    |         |    +--- :app:compileDebugKotlin
......
```


