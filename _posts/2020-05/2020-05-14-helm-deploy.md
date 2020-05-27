



# 编译docker image

之前大多是纸上谈兵，今天为了把serverbuild的bot部署到kubernetes，从头体验了一下通过helm部署的流程

第一步要把我们的bot代码编程一个docker image

```shell
[root@rain-working-vm autojob-bot]# tree .
.
├── autojob.py
├── cmc.py
├── Dockerfile
└── requirements.txt

0 directories, 4 files
```

* autojob.py是我们bot的代码
* cmc.py是依赖的库代码
* requirements.txt是依赖包，可以通过在Dockerfile中`RUN pip install -r requirements.txt`来安装依赖包
* Dockerfile是编辑docker image用的

然后可以开始编译并且上传

```shell
docker build -t registry-qa.webex.com/ocp/autojob-bot:v1.0.0 .
docker push registry-qa.webex.com/ocp/autojob-bot:v1.0.0
```

# 编写helm chart

之前有学习过helm char的编写[https://lrainsun.github.io/2019/11/30/kubernetes-helm/](https://lrainsun.github.io/2019/11/30/kubernetes-helm/)

```shell
[root@ocp-dev-003 autojob-bot]# tree .
.
├── Chart.yaml
├── templates
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   └── service.yaml
└── values.yaml

1 directory, 6 files
```

* Chart.yaml 版本和配置信息
* values.yaml 默认的配置
* templates 配置模板

# dry-run

在真正部署之前，我们可以测试一下，通过dry-run的方式

helm3强制要提供一个name，或者--generate-name

```shell
Usage:  helm install [NAME] [CHART] [flags]
helm install --dry-run -f autojob-conf/autojob.yaml --debug autojob-bot autojob-bot/ -n autojob-bot
```

autojob-conf/autojob.yaml是真正用的配置文件

因为chart包还没上传到helm repo，所以chart在这里是autojob-bot/，也就是本地的代码

可以检查真正生成的yaml文件跟预期的是不是一致

# 安装helm chart

真正安装的时候，去掉dry-run就可以了

```shell
helm install -f autojob-conf/autojob.yaml --debug autojob-bot autojob-bot/ -n autojob-bot
```

如果按照完了之后，我们想到有些配置需要改，只需要update就可以了

```shell
helm upgrade -f autojob-conf/autojob.yaml --debug autojob-bot autojob-bot/ -n autojob-bot
```

# 打包chart并上传

验证完了我们还可以打包chart并上传到chart repo

```shell
helm package autojob-bot/
helm plugin install https://github.com/chartmuseum/helm-push

helm repo add oke https://hlm.prv.webex.com/
helm plugin install https://github.com/chartmuseum/helm-push
helm push autojob-bot/ oke
helm repo update
helm search repo autojob
```

安装Helm chart from repo

```shell
helm install -f autojob-conf/autojob.yaml --debug autojob-bot oke/autojob -n autojob-bot
helm list -n autojob-bot
```

升级

```shell
helm upgrade -f autojob-conf/autojob.yaml --debug autojob-bot oke/autojob -n autojob-bot
```

