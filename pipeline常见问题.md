#### 一.指定某个stage阶段标记为跳过的方法

##### 1.1 方式一（声明式pipeline）

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
    allof {   ###and关系（我用着是不好使）
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
##### 1.2 方法二（脚本式pipeline）
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

#### 二.jenkins pipeline环境变量

##### 1.1 获取

方法一：展示所有默认环境变量，https:IP/env-vars.html
方法二：查看当前stage的所有环境变量，在pipeline中使用sh 'printenv | sort' 查看
方法三：sh "echo ${BUILD_NUMBER}"
方法四：sh "echo ${env.BUILD_NUMBER}"
方法五：sh "echo $BUILD_NUMBER"

##### 1.2 赋值

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

##### 1.3 覆盖

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

#### 三.sh的使用

##### 1.1 方法一

```
script{
    environment{
        sh_status1="${sh(script:'cmd',returnStdout:true)}"
    }
    sh_status2=sh(script:'cmd',returnStdout:true).trim()
    println $sh_status1
    println $sh_status2
    """
}
```

##### 1.2 方法二

```
sh 'echo nihao'
```

##### 1.3 方法三

```
sh """#!/bin/bash 
printenv
source "hello.sh"
"""
```

注意：在使用sh时，最好是简短的cmd命令，或直接调用shell文件，因为直接写大段的shell脚本在pipeline里，可能会遇到变量问题：

变量属于shell的变量，需要加\进行转义，eg:\\${a};如果取得是pipeline的env中变量，则不需要转义，直接${a}即可

#### 四.在一个jenkins pipeline中触发其他jenkins job

##### 1.1 一些参数在两个pipeline中传递

```
stage('sybervisor'){
    steps{
        script{
            def job_status=build job: '需要触发的job名字', propagate: false, wait: true, parameters: [string(name:'job的参数1', value:'值'), string(name:'job的参数2', value:'值')]
            println job_status.getProjectName()
            println job_status.getNumber()
            println job_status.getBuildVaribles()
            println job_status.buildVariables.var_name
        }
    }
}
```

getBuildVaribles()，buildVariables.变量名

这两个方法在使用时，得到的变量一定是以env环境变量方式定义才会被读取到

```
script {
    env.number = '2'
    env.var_name = "R001.0.0.${number}"
    env.build_status=currentBuild.currentResult   #这个方法可以得到stage的构建状态，返回成功或者失败；跟currentBuild.result的区别：result值可修改，currentResult不可修改,因为在代码正常执行完成时，currentResult总会返回SUCCESS,但实际运行结果是失败的（失败原因能用来catchError或者catch捕获了），这个时候我们就要手动将结果修改成FAILURE,所以用result比较合适
}
```

得到的结果是

```
Scheduling project:需要触发的job名字
Starting building:需要触发的job名字 #1
需要触发的job名字
1
{number=2,var_name=R001.0.0.2,build_status=SUCCESS}
R001.0.0.2
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

`还有更多方法可参考`:
`https://javadoc.jenkins.io/plugin/workflow-support/org/jenkinsci/plugins/workflow/support/steps/build/RunWrapper.html`

#### 五.并行stage

##### 1.1 方法一
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



##### 1.2 方法二

```
script {
    actionTest = loadList.collectEntries{
        name ->
                ["${name}": {
                    node("${name}") {  #注意这里是脚本式pipeline，注意不能使用post来获取stage的值
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

#### 六.option的使用

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

#### 七.agent用法

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
#### 八.pipeline 获取当前任务构建人
全局变量中默认不支持获取当前构建人，想要得到构建人信息，需要安装插件build-user-vars-plugin.hpi
```
# 下载插件的源码
wget https://github.com/jenkinsci/build-user-vars-plugin/archive/build-user-vars-plugin-1.5.zip

# 进入到解压后的插件目录中, 执行mvn打包命令
mvn clean package

# 完成后，在target目录中会生成一个build-user-vars-plugin.hpi文件
# 将.hpi结尾的文件，jenkins上手动上传插件即可

#Pipeline Examples
node {
  wrap([$class: 'BuildUser']) {
    def user = env.BUILD_USER_ID
  }
}

${currentBuild.durationString}".split("and counting")[0]  //获取任务持续时间
```

#### 九.pipeline 获取状态的接口

前缀：http://{jenkins网址}/{job_name}

##### 1.1 方式一

`/api/xml?pretty=true`

`/api/json?pretty=true`

`/api/python?pretty=true`

`/{job_num}/api/....`



##### 1.2 方式二

`/wfapi/runs`

`/wfapi/runs?fullStages=true  #可以详细到stageFlowNodes节点,但只能显示最新十条数据`

`/{job_num}/wfapi`



返回的json结构（因项目而异）

以`/wfapi/runs?fullStages=true`为例

- json以数组形式显示[]

  - 每次构建 为数组中的一个字典结构{}

    字典结构：_links{self},   id，name，**status**,  startTimeMillis,  endTimeMillis,   durationMillis,  pauseDuratuinMillis,   queueDurationMillis，stages

    - stages以数组形式显示[]

      - 每个stage 为数组的一个字典结构{}

        字典结构：_links{self},   id，name，execNode,   **status**,  startTimeMillis,  endTimeMillis,   durationMillis,  pauseDuratuinMillis,   stageFlowNodes

        - stageFlowNodes以数组形式显示[]
          - 每条命令 为数组的一个字典结构{}
          - 字典结构：_links{self,  log,  console},   id，name，execNode,   **status**,  startTimeMillis,  durationMillis,  pauseDuratuinMillis，  parentNodes

**status**：SUCCESS/FAILED/NOT_EXECUTED/IN_PROCESS/UNSTABLE

























