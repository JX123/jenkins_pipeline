﻿﻿#### 指定某个stage阶段标记为跳过的方法

##### 方式一（声明化pipeline）

```jenkins pipeline
when {
        expression {<some boolean expression>}  ###如果表达式的结果为false，则跳过该阶段
        }
```

eg:

```jenkins pipeline
stage("stage1"){
    when {
            environment name:'CHECK_STATUS',value:'0'  ###当环境变量'CHECK_STATUS'取值为0时，执行该stage，否则跳过
            }
    steps{
        sh 'echo skipped'
    }
}
```

当多条件组合判断，以；隔开

```
when {
    allof {   ###and关系
        branch 'master';
        environment name: 'DEPLOY_TO', value: 'production'
    }
}
```

```
when {
    anyof {   ### or关系
        branch 'master';
        environment name: 'DEPLOY_TO', value: 'production'
    }
}
```

如果碰到报错：java.lang.NoSuchMethodError:No such DSL method 'when' found among steps,可以使用方法二，具体原因不太清楚
##### 方法二（脚本化pipeline）
导入工具类：import org.jenkinsci.plugins.pipeline.modeldefinition.Utils

```
node(){
    stage("stage2"){
        script{
            if(true){
                echo '该stage被跳过'
                Utils.markStageSkippedForConditional("stage1")
            }else{
                echo '如果是false，将被执行'
            }
        }
}
```

注意：使用此种方法，请取消选中Use Groovy Sandbox复选框。

#### jenkins pipeline环境变量

##### 获取

方法一：展示所有默认环境变量，https:IP/env-vars.html
方法二：查看当前stage的所有环境变量，在pipeline中使用sh 'printenv | sort' 查看
方法三：sh "echo ${BUILD_NUMBER}"
方法四：sh "echo ${env.BUILD_NUMBER}"
方法五：sh "echo $BUILD_NUMBER"

##### 赋值

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

##### 覆盖

environment{}只能覆盖用environment{}赋值的变量
env.VAR='' 只能覆盖env.VAR='' 赋值的变量
withEnv(["env=value"]){}可以覆盖任何环境变量，作用范围只是{}内

关于环境变量值的问题：
默认环境变量的值均为字符串，即使false，true这种布尔值也会被转换成"false","true",如果想正确使用布尔值->"false".toBoolean()

```
...
environment{boo=false}
...
script{
    if(env.boo){echo "\"false\" 的判断时true，所以会被打印"}
    if(env.boo.toBoolean() == false){echo "false==false 为true，所以会被打印"}
}
...
```

#### sh的使用

##### 方法一

```
script{
    environment{
        sh_status1="${sh(script:'cmd',returnStdout:true)}"
    }
    sh_status2=sh(script:'cmd',returnStdout:true).trim()
    sh """#!/bin/bash
    echo $sh_status1
    echo $sh_status2
    """
}
```

##### 方法二

```
sh 'echo nihao'
```

##### 方法三

```
sh """#!/bin/bash 
printenv
source "hello.sh"
"""
```

#### 在一个jenkins pipeline中触发其他jenkins job

##### 一些参数在两个pipeline中传递

```
stage('sybervisor'){
    steps{
        script{
            def job_status=build job: '需要触发的job名字', propagate: false, wait: true, parameters: [string(name:'job的参数1', value:'值'), string(name:'job的参数2', value:'值')]
            println job_status.getProjectName()
            println job_status.getNumber()
            println job_status.getBuildVaribles()
            println job_status.buildVariables.变量名
        }
    }
}
```

##### 执行状态

```
stage('sybervisor'){
    steps{
        script{
            def job_status=build job: '需要触发的job名字', propagate: false, wait: true, parameters: [string(name:'job的参数1', value:'值'), string(name:'job的参数2', value:'值')]
            if(job_status.getResult() != "SUCCESS"){
                error "This pipeline stops here"
            }
            env.sybervisor='finish'
        }
    }
}
```

`还有更多方法可参考`
`https://javadoc.jenkins.io/plugin/workflow-support/org/jenkinsci/plugins/workflow/support/steps/build/RunWrapper.html`

#### 并行stage

##### 方法一
```
pipeline {
    agent {label 'test'}
    stages {
        stage('stage3') {
            failFast true  ###当并行任务有一个失败则流水线失败退出
            parallel{  ###并行配置，声明式书写方式
                stage('stage4'){
                    steps{}
                }
                stage('stage5'){
                     stages{
                         stage('stage6'){
                             steps{}   ###一个stage只能有一个steps
                        }
                    }
                }
            }
        }
    }
}
```



##### 方法二

```
script {
    actionTest = loadList.collectEntries{
        name ->
                ["${name}": {
                    node("${name}") {
                            stage("${name} stage") {
                                script{
                                    sh """#!/bin/bash
                                        set +ex
                                        echo "finish"
                                    """
                            	}
                           }
	    }
              }]
       } 
     parallel actionTest    ###并行配置，脚本式书写方式
}
```

`脚本式pipeline，可以使用grovvy语法`

#### option的使用

```
options{
        timestamps()    ### 在日志中显示时间
        timeout(time: 1, unit: 'HOURS')   ### 超过1小时，停止流水线,SECONDS（秒），MINUTES（分钟）可在steps中使用
        retry(3)   ###失败时重试3次，可在steps中使用
        disableConcurrentBuilds()   ### 不允许并行执行pipeline任务
        skipStageAfterUnstable()    ### 一旦构建状态进入了 unstable 状态就跳过此stage
        buildDiscarder(logRotator(numToKeepStr: '3'))     ### 保存最近历史构建记录的数量
}
```

```
steps{
        error("执行失败") ###抛出异常，中断整个pipeline
        sleep  200  ### 睡眠，默认秒
        waitUntil    ### 循环闭包操作，直到true退出
}
```

参考文档

[https://www.52wiki.cn/project-14/doc-614/]()

#### agent用法

##### 定义在pipeline顶层，用来设置执行节点

any：可以在任意可用的 agent上执行pipeline

```
agent any
```

none：pipeline将不分配全局agent，每个 stage分配自己的agent

```
agent none
```

label：指定运行节点agent的 Label

```
agent(label "slave_node")
```

node：自定义运行节点配置

```
agent{
  node{
    label "slave_node"
	customWorkspace "workspace"
   }
}
```

docker：使用给定的容器执行流水线。

```
agent{
	docker{
   		image '镜像名'
   		args ‘-v /etc:/etc’
   	}
}
```

dockerfile：使用源码库中包含的Dockerfile构建的容器来执行Pipeline；若要使用此项，必须从`Pipeline from SCM`加载

```
agent{dockerfile true}  ###Dockerfile 在源码根目录
```

```
agent{dockerfile {dir 'otherpath'}}   ###用dir指定dockerfile路径
```

使用additionalBuildArgs选项传递参数给docker build

```
agent{
	dockerfile{
		additionalBuildArgs '--build-arg a=b'
	}
}
```



