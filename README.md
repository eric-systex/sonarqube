 


# SonarQube - Continuous Code Quality
<kbd>![](https://i.imgur.com/DiZm5Pa.png)</kbd>

## SonarQube


### Introduction
SonarQube就是一套 Open Source 的程式碼品質分析工具。

SonarQube可以分為兩個部分，一個部分是負責執行程式碼分析的Runner，依語言種類有所不同，Sonar可支援25種以上的程式語言，如：Java、C#、JavaScript、Python…等等。

SonarJava是其中一個Runner，負責Java的程式碼分析，SonarJava提供540+個分析規則，其中包含124個用來識別Bug，341個用來識別code smells，57個用來識別漏洞。

這些規則可以檢測軟體品質相關的問題，例如：UNUSED CODE, LOGIC ERROR, CODING CONVENTION, PERFORMANCE HOTSPOT, RESOURCE LEAK, MULTI-THREADING, NULL-POINTER DEREFERENCE, ERROR HANDLING…等等。

可用來強制遵守以下幾個標準：

* CWE (Common Weakness Enumeration)
* CERT (Computer Emergency Response Team)
* OWASP TOP TEN (Open Web Application Security Project)
* SANS TOP 25

Core Components
* Plugins
* Rules
* Quality Profiles
* Quality Gates

### Install
透過 Helm 安裝至 EKS，安裝後，將會有

* SonarQube Server
* SonarQube Database (PostgreSQL)

```shell=
helm install stable/sonarqube \
    --set persistence.enabled=true \
    --set postgresql.persistence.storageClass=aws-efs \
    --set persistence.storageClass=aws-efs \
    --set persistence.accessMode=ReadWriteMany \
    --set persistence.size=1Gi \
    --name sonarqube \
    --namespace sonar
```

再將 sonarqube 服務對外 http://sonar.pluto.ezlab.xyz

```shell=
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sonarqube
spec:
  gateways:
  - devops-camp-gateway
  hosts:
  - sonar.pluto.ezlab.xyz
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: sonarqube-sonarqube.sonar.svc.cluster.local
        port:
          number: 9000
EOF
```

### Configuration
#### Disable SCM Sensor
進入 Administration > Configuration > General Settings > SCM 將之設為 True

<kbd>![](https://i.imgur.com/mjJJA3i.png)</kbd>


#### Force User Authentication
進入 Administration > Configuration > General Settings > Security 將之設為 True

<kbd>![](https://i.imgur.com/ABOVeoi.png)</kbd>


#### Create Webhook
每次 SonarQube Task 結束後，若需要將 Analysis 結果以 POST 方式通知外部系統（如：Jenkins），則需要設定 webhook

* Name: 任意字串
* URL: http://jenkins.default/sonarqube-webhook/  （sonarqube-webhook 為固定）
* Secret: 任意字串 

進入 Administration > Configuration > Webhooks > Create 新增 

<kbd>![](https://i.imgur.com/gCjoCME.png)</kbd>

<kbd>![](https://i.imgur.com/dCYHmIy.png)</kbd>


#### Create User for Jenkins
進入 Administration > Security > Create User 新增（只會用到此 user 的 token 來認證，密碼無用）

<kbd>![](https://i.imgur.com/w7HBQiA.png)</kbd>


新增後，點擊 Token，輸入任意字串，產生 token 並記錄之 

<kbd>![](https://i.imgur.com/5xhbkzG.png)</kbd>


#### Edit Global Permissions
全域的權限調整如下（依實際需求調整）主要是

* 開放 jenkins 的 Execute Analysis 權限 

進入 Administration > Security 調整

<kbd>![](https://i.imgur.com/k7sF0ez.png)</kbd>


#### Create Project
手動建立 SonarQube Project，輪入以下資料

* Name: api-fisc （必須與 Maven Artifact ID 相同）
* Key: com.systex.neptune:api-fisc（必須與 Maven Group ID + Artifact ID 相同）
* Visibility: Private

進入 Administration > Projects > Create Project

<kbd>![](https://i.imgur.com/v4vQfMF.png)</kbd>


新增後，點擊 Edit Permissions，權限開放調整如下（依實際需求調整）主要是

* 此專案開放 sonar-users 的 Browse 權限 

<kbd>![](https://i.imgur.com/jDTVscQ.png)</kbd>



#### Install Plugins
安裝以下 Plugins（依實際狀況安裝）

* ~~3D Code Metrics~~
* Checkstyle
* Code Smells
* Findbugs
* MyBatis Plugin for SonarQube
* SonarCSS
* SonarHTML
* SonarJS
* SonarJava
* SonarPython
* SonarTS

進入 Administration > Marketplace 安裝上述 plugins 後，重啟 Server

<kbd>![](https://i.imgur.com/QEtwvha.png)</kbd>


## Jenkins
### Credentials
總共需要新增以下 credentials

* sonar: 用來登入 SonarQube（SonarQube User Token）
* sonar-webhook: 用來同步 Analysis 的結果（SonarQube Project Webhook Secret）

![](https://i.imgur.com/61qjZFl.png)


進入 Credentials > global > Add credentials 新增 

#### SonarQube User Token
填入以下資料

* Kind: Secret Text
* Secret: (填入 User Token）
* ID: sonar

#### SonarQube Project Webhook Secret
填入以下資料

* Kind: Secret Text
* Secret: (填入 Webhook Secret）
* ID: sonar-webhook

### Plugins
#### Install Plugins
安裝 SonarQube Scanner 外掛

* SonarQube Scanner for Jenkins

<kbd>![](https://i.imgur.com/mspmBMx.png)</kbd>


#### Add SonarQube Scanner
~~進入 管理 Jenkins > Global Tool Configuration~~

~~按下 新增 SonarQube Scanner，勾選 自動安裝（Install from Maven Central）版本 4.2.0.1873~~

<kbd>![](https://i.imgur.com/T7mZauj.png)</kbd>


### SonarQube Server
#### Add SonarQube Server
進入 管理 Jenkins > 設定系統

按下 Add SonarQube 輸入以下資料，並勾選 Enable injection of SonarQube server configuration as build environment variables

* Name: sonar
* Server URL: http://sonar.pluto.ezlab.xyz

<kbd>![](https://i.imgur.com/O0tZ4ks.png)</kbd>


### Jobs
以 Neptune Fisc service 為例說明

* Java 
* Maven

#### Pipeline

```groovy=
node {
    def scm_credential = 'gitlab'
    def scm_repository = 'http://ailabgitlab.systex.com/neptune/api-fisc.git'
 
    stage ('checkout') {
        echo "♣♣♣ 自 SCM 取出源碼 branch: */master"
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], userRemoteConfigs: [[credentialsId: "${scm_credential}", url: "${scm_repository}"]]])
    }
 
    stage('SonarQube analysis') {
        echo "♣♣♣ SonarQube 源碼檢測"
        withSonarQubeEnv(installationName: 'sonar', credentialsId: 'sonar') {
            sh 'mvn clean package sonar:sonar'
        }
    }
}

// No need to occupy a node
stage("Quality Gate"){
    timeout(time: 10, unit: 'MINUTES') { // Just in case something goes wrong, pipeline will be killed after a timeout
        def qg = waitForQualityGate(credentialsId: 'sonar', webhookSecretId: 'sonar-webhook') // Reuse taskId previously collected by withSonarQubeEnv
        if (qg.status != 'OK') {
            error "Pipeline aborted due to quality gate failure: ${qg.status}"
        }
    }
}
```





## References
* https://github.com/helm/charts/tree/master/stable/sonarqube
* https://docs.sonarqube.org/
* https://jenkins.io/doc/pipeline/steps/sonar/
* https://tpu.thinkpower.com.tw/tpu/articleDetails/563
* https://rules.sonarsource.com/java
* test



 