
https://blog.gmem.cc/argo

工作流规格（Spec）由一系列的Argo模板组成，每个模板包含：
    可选的输入段
    可选的输出段
    容器调用声明，或者Step列表（每个Step调用其它容器模板）


# 1 Hello World

一个工作流的定义，就是一个K8S的CR：
```yaml	
apiVersion: argoproj.io/v1alpha1
# 新的资源类型
kind: Workflow
metadata:
  # 工作流名字规范
  generateName: hello-world-
spec:
  # 执行的PodTemplate名称
  entrypoint: whalesay
  templates:
    # 就是这个PodTemplate
  - name: whalesay
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["hello world"]
      # 可以加任何Pod的配置项
      resources:
        limits:
          memory: 32Mi
          cpu: 100m
```


# 2 参数化

简单修改一下上面的例子：

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-parameters-
spec:
  # 入口点
  entrypoint: whalesay
  # 调用入口点的参数
  arguments:
    parameters:
    - name: message
      value: hello world
 
  templates:
  - name: whalesay
    inputs:
      parameters:
      # 参数声明
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      # 参数引用
      args: ["{{inputs.parameters.message}}"]
```


提交此工作流时，可以指定实际参数值：
```
argo submit arguments-parameters.yaml -p message="goodbye world"
```

甚至，你还可以覆盖入口点，调用任何Pod Template：
```
argo submit arguments-parameters.yaml --entrypoint whalesay-caps
```


# 3 多Step工作流

你可以定义包含多个步骤的工作流，这些步骤可以并行、串行的执行：

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: steps-
spec:
  entrypoint: hello-hello-hello
  # 此工作流包含两个模板
  templates:
  - name: hello-hello-hello
    # 这个模板定义了工作流的步骤
    steps:
    # 步骤一
    - - name: hello1            
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello1"
    # 步骤二，它包含两个并行的步骤
      # 并行步骤1
    - - name: hello2a           
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello2a"
      # 并行步骤2
      - name: hello2b           
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello2b"
 
  # 这个模板负责执行工作流的步骤
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```

# 4 DAG工作流 

定义包含多个步骤的工作流的另外一种方式是，通过声明任务之间的依赖，产生DAG（有向无环图） 。在定义复杂工作流时，DAG更加简单、具有最大并发性。

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond-
spec:
  entrypoint: diamond
  templates:
  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]
  # 入口点模板
  - name: diamond
    # DAG声明
    dag:
      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        # 任务B依赖于任务A
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        # 任务D同时依赖于任务B、C
        dependencies: [B, C]
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
```

在DAG中，引用其它Task时使用tasks前缀，而非steps前缀：

```
{{tasks.generate-parameter.outputs.parameters.hello-param}}
```


# 5 使用构件

在工作流中，某些步骤产生或者消费构件，是很常见的需求。通常，前一环节的输出构件，用作下一环节的输入构件。 

下面的例子包含两个Step，前一个步骤产生构件供后一个消费：
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifact-passing-
spec:
  entrypoint: artifact-example
  templates:
  - name: artifact-example
    steps:
    # 产生构件
    - - name: generate-artifact
        template: whalesay
    # 消费构件
    - - name: consume-artifact
        template: print-message
        arguments:
          artifacts:
          # 绑定构件名message到generate0artifact输出的hello-art构件
          - name: message
            from: "{{steps.generate-artifact.outputs.artifacts.hello-art}}"
 
  # 此模板产生构件
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["cowsay hello world | tee /tmp/hello_world.txt"] 
    # 输出构件声明
    outputs:
      artifacts:
      - name: hello-art
        path: /tmp/hello_world.txt
 
  # 此模板消费构件
  - name: print-message
    # 输入构件声明
    inputs:
      artifacts:
      - name: message
        path: /tmp/message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["cat /tmp/message"]
```

# 6 Secret

Argo支持和K8S一致的保密字典格式，并且可以作为卷或环境变量挂载
```
# 使用kubectl创建保密字典
# kubectl create secret generic my-secret --from-literal=mypassword=S00perS3cretPa55word
 
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: secret-example-
spec:
  entrypoint: whalesay
  # 卷声明
  volumes:
  - name: my-secret-vol
    secret:
      secretName: my-secret
  templates:
  - name: whalesay
    container:
      image: alpine:3.7
      command: [sh, -c]
      args: ['
        echo "secret from env: $MYSECRETPASSWORD";
        echo "secret from file: `cat /secret/mountpath/mypassword`"
      ']
      # 作为环境变量访问
      env:
      - name: MYSECRETPASSWORD  # name of env var
        valueFrom:
          secretKeyRef:
            name: my-secret
            key: mypassword
      # 挂载为卷
      volumeMounts:
      - name: my-secret-vol
        mountPath: "/secret/mountpath"
```

# 7 脚本和result

很多情况下，我们仅仅希望Template来执行一个here-script： 
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: scripts-bash-
spec:
  entrypoint: bash-script-example
  templates:
  - name: bash-script-example
    steps:
    - - name: generate
        template: gen-random-int-bash
    - - name: print
        template: print-message
        arguments:
          parameters:
          - name: message
            # 引用此特殊的输出参数
            value: "{{steps.generate.outputs.result}}"
 
  - name: gen-random-int-bash
    # 在script关键字的source标签中，可以编写脚本
    # script还导致执行脚本时的标准输出，保存为名为result的特殊输出参数
    script:
      image: debian:9.4
      command: [bash]
      source: |
        # Shell脚本
        cat /dev/urandom | od -N2 -An -i | awk -v f=1 -v r=100 '{printf "%i\n", f + r * $1 / 65536}'
 
  - name: gen-random-int-python
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        # Python脚本
        import random
        i = random.randint(1, 100)
        print(i)
 
  - name: gen-random-int-javascript
    script:
      image: node:9.1-alpine
      command: [node]
      source: |
        # JS脚本
        var rand = Math.floor(Math.random() * 100);
        console.log(rand);
 
  - name: print-message
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo result was: {{inputs.parameters.message}}"]
```


# 8 输出参数

输出参数提供了一种（比构件）轻量的、使用Step结果的机制，你可以在任何类型的Step中使用输出参数，而不仅仅是上例中的script。
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: output-parameter-
spec:
  entrypoint: output-parameter
  templates:
  - name: output-parameter
    steps:
    - - name: generate-parameter
        template: whalesay
 
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["echo -n hello world > /tmp/hello_world.txt"]
    # 输出参数声明
    outputs:
      parameters:
      # 此参数来自文件
      - name: hello-param
        valueFrom:
          path: /tmp/hello_world.txt
```

# 9 循环

在单个Step中，迭代一组条目：
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-
spec:
  entrypoint: loop-example
  templates:
  - name: loop-example
    steps:
    - - name: print-message
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "{{item}}"
        # 针对以下两组输入，并行执行whalesay模板
        withItems: 
        - hello world
        - goodbye world
 
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```

条目可以是复杂对象： 
```
spec:
  entrypoint: loop-map-example
  templates:
  - name: loop-map-example
    steps:
    - - name: test-linux
        template: cat-os-release
        arguments:
          parameters:
          - name: image
            value: "{{item.image}}"
          - name: tag
            value: "{{item.tag}}"
        withItems:
        - { image: 'debian', tag: '9.1' } 
        - { image: 'debian', tag: '8.9' }
        - { image: 'alpine', tag: '3.6' }
        - { image: 'ubuntu', tag: '17.10' }
```



条目列表可以作为输入参数：
```
spec:
  entrypoint: loop-param-arg-example
  # 将条目列表定义为参数
  arguments:
    parameters:
    - name: os-list                                     
      value: |
        [
          { "image": "debian", "tag": "9.1" },
          { "image": "debian", "tag": "8.9" },
          { "image": "alpine", "tag": "3.6" },
          { "image": "ubuntu", "tag": "17.10" }
        ]
 
  templates:
  - name: loop-param-arg-example
    inputs:
      parameters:
      - name: os-list
    steps:
    - - name: test-linux
        template: cat-os-release
        arguments:
          parameters:
          - name: image
            value: "{{item.image}}"
          - name: tag
            value: "{{item.tag}}"
        # 引用参数，针对每个元素，执行test-linux容器
        withParam: "{{inputs.parameters.os-list}}" 
```

# 10 分支

下面是一个抛硬币的例子，根据结果是正还是反，决定下一Step：

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: coinflip-
spec:
  entrypoint: coinflip
  templates:
  - name: coinflip
    steps:
    # 前置步骤
    - - name: flip-coin
        template: flip-coin
    # 分支步骤
    - - name: heads
        template: heads
        # 如果上以步骤的输出结果为heads
        when: "{{steps.flip-coin.outputs.result}} == heads"
      - name: tails
        template: tails                 # call tails template if "tails"
        # 如果上以步骤的输出结果为tails
        when: "{{steps.flip-coin.outputs.result}} == tails"
 
  # Return heads or tails based on a random number
  - name: flip-coin
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        result = "heads" if random.randint(0,1) == 0 else "tails"
        print(result)
 
  - name: heads
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was heads\""]
 
  - name: tails
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was tails\""]
```


# 11 递归

Argo模板甚至可以递归的相互调用，下面的例子，直到抛硬币结果是正面，才会结束工作流：

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: coinflip-recursive-
spec:
  entrypoint: coinflip
  templates:
  - name: coinflip
    steps:
    # 翻转硬币
    - - name: flip-coin
        template: flip-coin
    # 并行估算结果
    - - name: heads
        template: heads
        when: "{{steps.flip-coin.outputs.result}} == heads"
      - name: tails
        # 如果结果是反面，则递归的调用coinflip模板
        template: coinflip
        when: "{{steps.flip-coin.outputs.result}} == tails"
```


# 12 退出处理器

Exit handler是一种必然会在工作流结尾执行的模板，不论工作流执行成功与否。它的运用场景包括：

    清理
    发送工作流状态的通知
    将成功/失败状态传递为Webhook结果（例如GitHub Build Result）
    重新提交工作流
    提交另外一个工作流

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: exit-handlers-
spec:
  entrypoint: intentional-fail
  # 声明退出处理器
  onExit: exit-handler
  templates:
  # 工作流主模板
  - name: intentional-fail
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo intentional failure; exit 1"]
 
  # 退出处理器模板
  # 主模板完成后，工作流状态可以通过全局变量{{workflow.status}}获取，其值是Succeeded, Failed, Error之一
  - name: exit-handler
    steps:
    - - name: notify
        template: send-email
      - name: celebrate
        template: celebrate
        when: "{{workflow.status}} == Succeeded"
      - name: cry
        template: cry
        when: "{{workflow.status}} != Succeeded"
  - name: send-email
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo send e-mail: {{workflow.name}} {{workflow.status}}"]
  - name: celebrate
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo hooray!"]
  - name: cry
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo boohoo!"]
```


# 13 超时

可以限制工作流中Step执行的最大时长：
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: timeouts-
spec:
  entrypoint: sleep
  templates:
  - name: sleep
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo sleeping for 1m; sleep 60; echo done"]
    # 最大10秒超时
    activeDeadlineSeconds: 10
```

# 14 持久卷

可以再工作流中声明VPC并挂载：

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: volumes-pvc-
spec:
  entrypoint: volumes-pvc-example
  # 声明PVC
  volumeClaimTemplates:
  - metadata:
      name: workdir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
 
  templates:
  - name: volumes-pvc-example
    steps:
    - - name: generate
        template: whalesay
    - - name: print
        template: print-message
 
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["echo generating message in volume; cowsay hello world | tee /mnt/vol/hello_world.txt"]
      # 挂载为卷
      volumeMounts:
      - name: workdir
        mountPath: /mnt/vol
 
  - name: print-message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo getting message from volume; find /mnt/vol; cat /mnt/vol/hello_world.txt"]
      # 挂载为卷
      volumeMounts:
      - name: workdir
        mountPath: /mnt/vol
```


# 15 守护容器 

这种容器，在工作流执行期间，运行于后台，并且在工作流退出时自动销毁。

```
spec:
  entrypoint: daemon-example
  templates:
  - name: influxdb
    # 启动为守护容器
    daemon: true
    container:
      image: influxdb:1.2
```


# 16 Sidecar

用于创建具有多个容器的Pod：
```
spec:
  entrypoint: sidecar-nginx-example
  templates:
  - name: sidecar-nginx-example
    container:
      image: appropriate/curl
      command: [sh, -c]
      args: ["until `curl -G 'http://127.0.0.1/' >& /tmp/out`; do echo sleep && sleep 1; done && cat /tmp/out"]
    sidecars:
    - name: nginx
      image: nginx:1.13
```

需要注意容器启动的顺序是随机的，因此例子中的主容器需要轮询，直到sidecar准备好，再执行主逻辑。 


# 17 内置构件类型

Argo内置了对Git、HTTP、S3等构件的支持： 

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hardwired-artifact-
spec:
  entrypoint: hardwired-artifact
  templates:
  - name: hardwired-artifact
    inputs:
      artifacts:
      - name: argo-source
        path: /src
        # 来自Git的输入构件，签出项目到src目录下
        git:
          repo: https://github.com/argoproj/argo.git
          # 可以指定分支、Commit、Tag等
          revision: "master"
      - name: kubectl
        path: /bin/kubectl
        mode: 0755
        # 来自HTTP构件，下载到/bin目录下并且修改为可执行
        http:
          url: https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl
      - name: objects
        path: /s3
        # 来自s3的构件，拷贝一个桶中的键值并且存放到s3目录
        s3:
          endpoint: storage.googleapis.com
          bucket: my-bucket-name
          key: path/in/bucket
          accessKeySecret:
            name: my-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-s3-credentials
            key: secretKey
    container:
      image: debian
      command: [sh, -c]
      args: ["ls -l /src /bin/kubectl /s3"]
```

# 18 管理K8S资源

某些情况下需要再Argo工作流中管理K8S资源，可以通过资源模板实现K8S资源的增删改：

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: k8s-jobs-
spec:
  entrypoint: pi-tmpl
  templates:
  - name: pi-tmpl
    # 提示这是一个资源模板
    resource:             
      # 支持任何kubectl动作，例如create, delete, apply, patch
      action: create
      # 可选的成功条件、失败条件表达式
      # 如果failureCondition=true，认为此Step失败
      # 如果successCondition=true，认为此Step成功
      #  
      # 表达式使用K8S标签选择器语法，但是可以用于任何字段（不单单是label）
      # 多个表达式用逗号分隔，表示逻辑与
      successCondition: status.succeeded > 0
      failureCondition: status.failed > 3
      # 资源的定义
      manifest: |
        apiVersion: batch/v1
        kind: Job
        metadata:
          generateName: pi-job-
        spec:
          template:
            metadata:
              name: pi
            spec:
              containers:
              - name: pi
                image: perl
                command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
              restartPolicy: Never
          backoffLimit: 4
```



    创建的K8S资源独立于工作流，也就是说工作流完成后资源仍然存在
    如果希望资源在工作流被删除时，级联删除，可以使用ownerReference
    对于patch操作，支持额外的属性mergeStrategy，取值：
        strategic，默认值。CR不支持strategic
        merge
        json
    在资源定义清单中你可以使用变量，就像Helm Chart一样



# 19 DinD

为了在容器中执行docker命令，需要使用Docker in Docker：

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: sidecar-dind-
spec:
  entrypoint: dind-sidecar-example
  templates:
  - name: dind-sidecar-example
    container:
      image: docker:17.10
      command: [sh, -c]
      args: ["until docker ps; do sleep 3; done; docker run --rm debian:latest cat /etc/os-release"]
      env:
      # Dockerd所在主机名（当前Pod）
      - name: DOCKER_HOST
        value: 127.0.0.1
    sidecars:
    - name: dind
      # Docker官方提供了运行Dockerd的镜像
      image: docker:17.10-dind
      securityContext:
        # DinD必须以特权容器的方式运行
        privileged: true
      # 将主容器中的卷，全部挂载到Sidecar中（包括构件）的相同路径
      mirrorVolumeMounts: true
```

# 20 自定义变量引用

默认情况下Argo仅仅识别"item", "steps", "inputs", "outputs", "workflow", "tasks"前缀的变量。


# 21 持续集成

```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: influxdb-ci-
 
spec:
  entrypoint: influxdb-ci
  # 参数定义
  arguments:
    parameters:
    # 版本库以及修订版
    - name: repo
      value: https://github.com/influxdata/influxdb.git
    - name: revision
      value: 1.6
 
  templates:
  - name: influxdb-ci
    # 步骤列表
    steps:
    # 1. 签出
    - - name: checkout
        template: checkout
    # 2.1 构建
    - - name: build
        template: build
        arguments:
          artifacts:
          - name: source
            from: "{{steps.checkout.outputs.artifacts.source}}"
    # 2.2 单元测试
      - name: test-unit
        template: test-unit
        arguments:
          artifacts:
          - name: source
            from: "{{steps.checkout.outputs.artifacts.source}}"
    # 3.1 覆盖率测试
    - - name: test-cov
        template: test-cov
        arguments:
          artifacts:
          - name: source
            from: "{{steps.checkout.outputs.artifacts.source}}"
    # 3.2 端到端测试
      - name: test-e2e
        template: test-e2e
        arguments:
          artifacts:
          - name: influxd
            from: "{{steps.build.outputs.artifacts.influxd}}"
 
  # 签出模板，很简单，直接声明输入、输出构件即可
  - name: checkout
    inputs:
      artifacts:
      - name: source
        path: /src
        git:
          repo: "{{workflow.parameters.repo}}"
          revision: "{{workflow.parameters.revision}}"
    outputs:
      artifacts:
      - name: source
        path: /src
    container:
      image: golang:1.9.2
      command: ["/bin/sh", "-c"]
      args: ["cd /src && git status && ls -l"]
 
  # 构建模板
  - name: build
    inputs:
      artifacts:
      - name: source
        path: /go/src/github.com/influxdata/influxdb
    outputs:
      artifacts:
      - name: influxd
        path: /go/bin
    container:
      image: golang:1.9.2
      # 通过Shell调用go命令进行构建
      command: ["/bin/sh", "-c"]
      args: ["
        cd /go/src/github.com/influxdata/influxdb &&
        go get github.com/golang/dep/cmd/dep &&
        dep ensure -vendor-only &&
        go install -v ./...
      "]
      resources:
        requests:
          memory: 1024Mi
          cpu: 200m
  # 单元测试模板
  - name: test-unit
    inputs:
      artifacts:
      - name: source
        path: /go/src/github.com/influxdata/influxdb
    container:
      image: golang:1.9.2
      command: ["/bin/sh", "-c"]
      args: ["
        cd /go/src/github.com/influxdata/influxdb &&
        go get github.com/golang/dep/cmd/dep &&
        dep ensure -vendor-only &&
        # 测试所有用例
        go test -parallel=1 ./...
      "]
 
  # 覆盖率测试模板
  - name: test-cov
    inputs:
      artifacts:
      - name: source
    steps:
    - - name: test-cov-query
        template: test-cov-base
        arguments:
          parameters:
          - name: package
            value: "query"
          artifacts:
          - name: source
            from: "{{inputs.artifacts.source}}"
      - name: test-cov-tsm1
        template: test-cov-base
        arguments:
          parameters:
          - name: package
            value: "tsdb/engine/tsm1"
          artifacts:
          - name: source
            from: "{{inputs.artifacts.source}}"
  - name: test-cov-base
    inputs:
      parameters:
      - name: package
      artifacts:
      - name: source
        path: /go/src/github.com/influxdata/influxdb
    outputs:
      artifacts:
      - name: covreport
        path: /tmp/index.html
    container:
      image: golang:1.9.2
      command: ["/bin/sh", "-c"]
      args: ["
        cd /go/src/github.com/influxdata/influxdb &&
        go get github.com/golang/dep/cmd/dep &&
        dep ensure -vendor-only &&
        go test -v -coverprofile /tmp/cov.out ./{{inputs.parameters.package}} &&
        go tool cover -html=/tmp/cov.out -o /tmp/index.html
      "]
      resources:
        requests:
          memory: 4096Mi
          cpu: 200m
  # 端到端测试模板
  - name: test-e2e
    inputs:
      artifacts:
      - name: influxd
    steps:
    - - name: influxdb-server
        template: influxdb-server
        arguments:
          artifacts:
          - name: influxd
            from: "{{inputs.artifacts.influxd}}"
    - - name: initdb
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: curl -XPOST 'http://{{steps.influxdb-server.ip}}:8086/query' --data-urlencode "q=CREATE DATABASE mydb"
    - - name: producer1
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: for i in $(seq 1 20); do curl -XPOST 'http://{{steps.influxdb-server.ip}}:8086/write?db=mydb' -d "cpu,host=server01,region=uswest load=$i" ; sleep .5 ; done
      - name: producer2
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: for i in $(seq 1 20); do curl -XPOST 'http://{{steps.influxdb-server.ip}}:8086/write?db=mydb' -d "cpu,host=server02,region=uswest load=$((RANDOM % 100))" ; sleep .5 ; done
      - name: producer3
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: curl -XPOST 'http://{{steps.influxdb-server.ip}}:8086/write?db=mydb' -d 'cpu,host=server03,region=useast load=15.4'
      - name: consumer
        template: influxdb-client
        arguments:
          parameters:
          - name: cmd
            value: curl --silent -G http://{{steps.influxdb-server.ip}}:8086/query?pretty=true --data-urlencode "db=mydb" --data-urlencode "q=SELECT * FROM cpu"
 
  # 守护容器
  - name: influxdb-server
    inputs:
      artifacts:
      - name: influxd
        path: /app
    daemon: true
    outputs:
      artifacts:
        - name: data
          path: /var/lib/influxdb/data
    container:
      image: debian:9.4
      readinessProbe:
        httpGet:
          path: /ping
          port: 8086
        initialDelaySeconds: 5
        timeoutSeconds: 1
      command: ["/bin/sh", "-c"]
      args: ["chmod +x /app/influxd && /app/influxd"]
      resources:
        requests:
          memory: 512Mi
          cpu: 250m
 
  # influxdb客户端
  - name: influxdb-client
    inputs:
      parameters:
      - name: cmd
    container:
      image: appropriate/curl:latest
      command: ["/bin/sh", "-c"]
      args: ["{{inputs.parameters.cmd}}"]
```

需要注意，例子中定义的模板很多，但仅仅入口点的steps中引用的模板才会主动实例化。





