### Jenkins pipeline

#### 基本概念

##### 定义

就是一套运行于Jenkins上的工作流框架，将原本独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排与可视化。

##### 优势

- 更好的版本化：将pipeline提交到版本库中进行版本控制
- 更好地协作：pipeline的每次修改对所有人都是可见的。除此之外，还可以对pipeline进行代码审查
- 更好的重用性：手动操作没法重用，但是代码可以重用

##### 官方手册

https://www.jenkins.io/doc/book/pipeline/

##### 使用方法

安装pipeline插件

##### 表现形式

Jenkinsfile有2种方式，第一种是可以直接在web配置中进行编写，这样只适合临时项目调试或简短的内容；

![image-20220906133916557](C:\Users\jiangxin\AppData\Roaming\Typora\typora-user-images\image-20220906133916557.png)

第二种是在远程仓库上进行管理，这里配置远程仓库地址，让job每次执行的时候拉取这个项目，然后执行其中的某个文件。

![image-20220906133932735](C:\Users\jiangxin\AppData\Roaming\Typora\typora-user-images\image-20220906133932735.png)

##### 语法

从2.5版本开始，同时支持两种格式的语法-脚本式（Scripted）语法和声明式（Declar-ative）语法，推荐使用声明式语法，它的使用人群更广泛，也更好表达维护；脚本式语法比较灵活，编写清晰简单，groovy的语法可以直接使用套用。

###### 声明式语法

```pipeline
pipeline{
    agent any
    stages{
        stage("first stage"){
            steps("first steps"){
                echo "this is first step"
            }
        }
    }
    post{
        always{
            echo "this is ending..."
        }
    }
}
```

1.pipeline :声明其内容为一个声明式的pipeline脚本

```
作用域：应用于全局最外层，表明该脚本为声明式pipeline
是否必须：必须
参数：无
```

2.agent:执行节点（job运行的slave或者master节点）

```
作用域：可用在全局与stage内
是否必须：是，
参数：any,none, label, node,docker,dockerfile
```

3.stages:阶段集合，包裹所有的阶段（例如：打包，部署等各个阶段）

```
作用域：全局或者stage阶段内，每个作用域内只能使用一次
是否必须：全局必须
参数：无
```

4.stage:阶段，被stages包裹，一个stages可以有多个stage

```
作用域：被stages包裹，作用在自己的stage包裹范围内
是否必须：必须
参数：需要一个string参数，表示此阶段的工作内容
备注：stage内部可以嵌套stages，内部可单独制定运行的agent
```

5.steps:步骤,为每个阶段的最小执行单元,被stage包裹

```
作用域：被stage包裹，作用在stage内部，每个作用域只能使用一次
是否必须：必须
参数：无
```

6.post:执行构建后的操作，根据构建结果来执行对应的操作

```
作用域：作用在pipeline结束后者stage结束后
条件：always、changed、failure、success、unstable、aborted
```

常用指令

**options指令：**

- buildDiscarder:指定build history与console的保存数量
  用法：options { buildDiscarder(logRotator(numToKeepStr: ‘1’)) }
- disableConcurrentBuilds：设置job不能够同时运行
  用法：options { disableConcurrentBuilds() }
- skipDefaultCheckout：跳过默认设置的代码check out
  用法：options { skipDefaultCheckout() }
- skipStagesAfterUnstable:一旦构建状态变得UNSTABLE，跳过该阶段
  用法：options { skipStagesAfterUnstable() }
- checkoutToSubdirectory:在工作空间的子目录进行check out
  用法：options { checkoutToSubdirectory(‘children_path’) }
- timeout:设置jenkins运行的超时时间，超过超时时间，job会自动被终止
  用法：options { timeout(time: 1, unit: ‘MINUTES’) }
- retry :设置retry作用域范围的重试次数
  用法：options { retry(3) }
- timestamps:为控制台输出增加时间戳
  用法：options { timestamps() }

```
options {
    timestamps() 
    disableConcurrentBuilds()
    }
```

environment指令：

```
environment {
    P1="parameters 1"
    }
```

parameters指令：

```
parameters {
    string(name: 'P1', defaultValue: 'it is p1', description: 'it is p1')
    booleanParam(name: 'P2', defaultValue: true, description: 'it is p2')
    }
```

**triggers指令：**

第一种：cron

- 作用：以指定的时间来运行pipeline
- 用法：triggers { cron(‘*/1 * * * *’) }

第二种：pollSCM

- 作用：以固定的时间检查代码仓库更新（或者当代码仓库有更新时）自动触发pipeline构建
- 用法：triggers { pollSCM(‘H */4 * * 1-5’) }或者triggers { pollSCM() }（后者需要配置post-commit/post-receive钩子）

第三种：upstream

- 作用：可以利用上游Job的运行状态来进行触发
- 用法：triggers { upstream(upstreamProjects: ‘job1,job2’, threshold: hudson.model.Result.SUCCESS) }

```
triggers { upstream(upstreamProjects: 'test_8,test_7', threshold: hudson.model.Result.SUCCESS) }
```

**env指令：**

方法一：

```
script{
    env.CHECK_STATUS='0'
}
```

方法二：

```
environment{
    CHECK_STATUS='0'
}
```

方法三：

```
withEnv(["CHECK_STATUS=0"]){
    echo "只在大括号范围内有效"
}
```

##### 

**tools指令：**

引用的工具需要在管理页面的全局工具配置里配置过

```
tools {
    maven 'apache-maven-3.0.1' 
    }
```

**input指令：**

允许暂时中断pipeline执行，等待用户输入，根据用户输入进行下一步动作

```
input {
    message "Should we continue?"
    ok "Yes, we should."
    submitter "alice,bob"
    parameters {
    string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    }
}
```

**when指令：**

根据when指令的判断结果来决定是否执行后面的阶段，如果我们想要在进入agent之前进行判断，需要将beforeAgent设置为true

- branch ：判断分支名称是否符合预期
  用法：when { branch ‘master’ }
- environment ： 判断环境变量是否符合预期
  用法：when { environment name: ‘DEPLOY_TO’, value: ‘production’ }
- expression：判断表达式是否符合预期
  用法：when { expression { return params.DEBUG_BUILD } }
- not : 判断条件是否为假
  用法：when { not { branch ‘master’ } }
- allOf：判断所有条件是不是都为真
  用法：when { allOf { branch ‘master’; environment name: ‘DEPLOY_TO’, value: ‘production’ } }
- anyOf：判断是否有一个条件为真
  用法：when { anyOf { branch ‘master’; branch ‘staging’ } }

```
when {
    beforeAgent true //设置先对条件进行判断，符合预期才进入steps
    branch 'production'
    }
```

**parallel命令：**

- 一个stage只能有一个steps或者parallel
- 嵌套的stages里不能使用parallel
- parallel不能包含agent或者tools
- 通过设置failFast 为true表示：并行的job中如果其中的一个失败，则终止其他并行的stage

```pipeline

stage('Run Tests') {
    failFast true
    parallel {
        stage('Test On Chrome') {
            agent { label "chrome" }
            steps {
            echo "Chrome UI测试"
            }
        }
        stage("Test On Firefox") {
            agent { label "firefox" }
            steps {
            echo "Firefox UI测试"
            }
        }
        stage("Test On IE") {
            agent { label "ie" }
            steps {
            echo "IE UI测试"
            }
        }
    }
}
```

**script命令：**

在声明式的pipeline中默认无法使用脚本语法，但是pipeline提供了一个脚本环境入口：script{},通过使用script来包裹脚本语句，即可使用脚本语法

```pipeline
steps {
    script{
        if ( "1" =="1" ) {
        echo "lalala"
        }else {
        echo "oooo"
        }
  	}
}
```

**异常处理命令：**

```pipeline
script{
    try {
        sh 'exit 1'
        }
    catch (exc) {
        echo 'Something failed'
        }
}
```

**获取sh命令结果：**

```pipeline
def status = sh(returnStatus: true, script: "git merge --no-edit $branches > merge_output.txt")
//returnStatus：true 返回0或者非0的整数

def status = sh(returnStdout: true, script: "git merge --no-edit $branches > merge_output.txt")
//returnStdout 输出命令结果
```

**触发其他job指令：**

```pipeline
build job: 'Test', parameters: [string(name: 'Name', value: param)], wait:true
```



###### 脚本式语法

node可以嵌套stage，也可以被stage嵌套

```
node {
    dir('/home/share/node/falcon') {
        stage("git") {
            sh "git fetch origin"
            sh "git checkout -f origin/master"
        }
}
```





























