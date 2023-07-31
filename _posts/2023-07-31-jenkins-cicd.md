---
title:  "Jenkins와 Nginx를 통한 무중단 배포"
categories:
  - backend
tags:
  - Jenkins
  - CICD
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">작성자: 이다빈</span></div>

# 🤔 왜 무중단 배포를 해야할까?

**무중단 배포(Zero-downtime deployment)**란 소프트웨어 또는 웹 애플리케이션을 업데이트하거나 새로운 버전을 배포할 때 중단 없이 서비스를 계속 제공하는 배포 방식을 말하며 기존의 서비스가 동작하면서 새로운 업데이트가 이루어지기 때문에 사용자들은 전환 과정에서 서비스 중단을 경험하지 않게 됩니다. 

우리 Explorer 스쿼드의 서비스 구조는 크게 아래 3가지로 나눌 수 있습니다.

- 메인넷 데이터를 실시간으로 ElasticSearch에 적재하는 Collector 서비스
- 적재한 데이터를 사용자 화면에 보여주기 위한 API 서비스
- 적재한 데이터를 추가적으로 가공하기 위해 일정 시간마다 돌아가는 Scheduler 서비스

이 중 하나라도 새로운 버전의 서비스가 배포되는 동안 서버가 멈추게 된다면 데이터 정합성 문제나 유저가 보고있는 화면등에 데이터가 노출되지 않는 등 문제가 생길 것 입니다. 

# 🤯  어떤 방법으로 무중단 배포를 할 수 있을까?

## ☝️ 로드 밸런싱

![출처 : Nginx](https://www.nginx.com/wp-content/uploads/2014/07/what-is-load-balancing-diagram-NGINX.png)

출처 : Nginx

로드 밸런싱은 여러대의 서버로 들어오는 트래픽을 분산시켜주는 방법입니다. 서버 그룹 간 트래픽을 고르게 분배하여 서버의 부하를 분산시킬 수 있습니다. 무중단 배포를 위해 새로운 버전의 애플리케이션을 배포한 서버 그룹과 기존 버전의 애플리케이션을 동작시키는 서버 그룹을 구성하고, 로드 밸런서 설정을 변경하여 새로운 서버 그룹으로 트래픽을 전환하는 방식을 사용할 수 있습니다.

## ✌️ 롤링 업데이트

![rolling.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/1-rolling.png)

롤링 업데이트는 서버 그룹을 순차적으로 업데이트 하며 새로운 버전의 애플리케이션을 배포하는 방법입니다. 기존 서버 그룹에서 하나씩 서버를 업데이트 하고, 업데이트 된 서버가 안정적으로 동작하는지 확인 한 후에 다음 서버를 업데이트 하는 방식을 반복하여 서비스 중단 없이 배포하는 방법입니다.

## 🤟 Blue-Green 배포

![blue.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/2-blue.png)

![green.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/3-green.png)

기존 환경(Blue)와 새로운 환경(Green)을 구성하고 기존 환경에서는 현재 서비스가 동작하고 있는 상태를 유지하며, 새로운 환경에 신규 버전의 애플리케이션을 배포하고 프록시 설정이나 로드밸런서 설정을 변경하여 트래픽을 기존 환경에서 새로운 환경으로 전환하여 서비스 중단 없이 배포하는 방법입니다.

# 😤 우리가 사용한 Blue/Green 배포 방식

우리의 개발 환경은 현재 1대의 서버로 구성되어 있어 AWS 로드밸런서를 통해 새로운 서버를 띄워 트래픽을 분산하는 방식은 적합하지 않았습니다. 또한 롤링 업데이트 같은 경우에는 구버전과 신버전의 어플리케이션이 동시에 서비스 되기 때문에 데이터 수집이나 업데이트 중 정합성 문제가 발생할 수 있다는 점에서 우리의 서비스 구성 방식과는 어울리지 않다고 판단했습니다.

따라서 우리는 Blue-Green 배포 방식을 채택하되, 1대의 서버에서 Blue / Green을 서버가 아닌 포트로 나누어 Nginx 프록시 방향을 달리하는 방법으로 무중단 배포 방식을 구성하기로 결정했습니다.

## 👀 Blue-Green 배포 구조

![배포프로세스.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/4-process.png)

구성한 배포 흐름은 아래와 같습니다. 개발용 서버이므로 배포 서버 1대, 프록시 서버 1대, Jenkins 서버 1대로 구성되어 있습니다.

1. 젠킨스가 해당 브랜치의 코드를 받아 jar 파일을 Build 한다.
2. Active 상태가 아닌 포트를 확인한다. (Backup port)
3. 배포 대상이 될 Backup 포트를 사용하고 있는 프로세스를 중지한다.
4. 신규 jar 파일을 배포 서버로 복제한다.
5. 서버에서 신규 jar 파일을 이용하여 프로세스를 시작한다.
6. 신규 프로세스 정상 실행 확인한다. (Health Check)
7. 정상 실행이 확인되면 Nginx 서버에서 revers proxy 대상 포트를 신규 포트로 변경하고 Nginx 서버를 reload한다.

프록시 방향이 변경되면, 기존에 Active 상태의 포트는 Backup용 포트가 됩니다. 만약 빌드 파일의 코드에 문제가 있다면 Nginx가 이를 감지하고 프록시 방향을 Backup용 포트로 바꿔줄 것입니다.

## ✅ Jenkins 설정

### Global 변수 설정

![스크린샷 2023-07-26 오후 11.54.11.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/5-global.png)

`Dashboard` > `Manage Jenkins` > `System` > `Global properties`

Pipeline에서 사용할 전역 변수를 생성할 수 있습니다. 이 변수들은 pipeline 내에서 `${Name}` 형식으로 사용할 수 있습니다.

### Credentials 생성

![스크린샷 2023-07-26 오후 11.59.29.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/6-credentials.png)

`Dashboard` > `Manage Jenkins` > `System` > `Credentials` > `Global credentials` > `Add Credentials` 
Github에서 발급받은 토큰을 입력합니다. 해당 토큰을 사용하여 Jenkins 서버가 대상 프로젝트를 clone 받을 수 있습니다.

### Pipeline 생성

![스크린샷 2023-07-26 오전 9.53.55.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/7-pipeline.png)

`Dashboard` > `New Item` 을 클릭하여 배포 파이프라인을 생성할 수 있습니다.

**Github 프로젝트 연결** 

Github project를 체크하고 파이프라인을 생성할 github project url을 입력합니다.

**파라미터 설정** 

![스크린샷 2023-07-26 오후 11.50.58.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/8-parameters.png)

`General` → `This project is parameterized` 체크 → `Add Parameter` → `String Prameter` 를 선택합니다.
Explorer 스쿼드에서는 프로젝트의 브랜치 별 배포를 위한 `branch` 파라미터를 사용하고 있습니다. 해당 변수는 Pipeline 안에서 `${branch}` 로 사용 가능합니다.

**Build Triggers 설정**

![스크린샷 2023-07-26 오전 10.05.44.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/9-build-triggers.png)

`Build Triggers` > `GitHub hook trigger for GITScm polling` 체크
위 기능은 Github에 연결한 Webhook trigger를 받으면 빌드를 실행합니다. 사전에 Github 프로젝트에 webhook 연동이 필요합니다.

## 🔨 Jenkins Pipeline

### 환경 변수 설정

```groovy
environment {
      SLACK_CHANNEL = '{slack channel}'
      registry = "{project registry}"
      JAR_FILE_PATH = '/data/jenkins/workspace/{pipeline name}/build/libs'
      DEPLOY_PATH = '/home/ubuntu/{project name}'
  }
```

파이프라인 상단에 해당 파이프라인에서만 사용할 변수를 지정할 수 있습니다. 배포 시작 / 성공 / 실패 시 알림받을 채널 이름, registry, Jenkins 서버의 jar 파일 위치, jar 파일을 배포 서버로 보낼 위치를 지정하였습니다.

### Branch checkout

```groovy
stage('Checkout') {                                                            # 빌드 시 입력한 branch parameters 값을 받아와서 소스코드 체크아웃 수행
    steps {
        git branch: "${branch}",
        credentialsId: 'credential id',
        url: 'github project url'
    }
}
```

입력받은 브랜치의 코드를 받기 위해서 branch를 checkout 하는 과정이 필요합니다. 사전에 Jenkins Credential에 등록한 Github 토큰이 여기서 사용됩니다. url은 clone 받을 때 사용하는 `~.git` url을 입력합니다.

### 버전 확인

```groovy
stage('Read Build Info') {                                                     # 생성할 jars 파일의 버전 확인
    steps {
            script {
                def versionFile = readFile 'build.gradle'
                def versionMatch = versionFile =~ /version\s+=\s+['"](.+)['"]/

                VERSION = versionMatch ? versionMatch[0][1] : 'Unknown'
                echo "Build Version: ${VERSION}"
            }
        }
    }
```

배포할 코드의 버전을 확인합니다. 

### 빌드

```groovy
stage('Build') {                                                              # gradlew 명령으로 빌드 실행
     steps {
        sh "./gradlew clean bootJar"
    }
}
```

여기서 중요한것은 github에 `gradle/wrapper/gradle-wrapper.jar` 파일이 올라가 있어야지만 jenkins 서버가 pull을 받고 빌드를 실행할 수 있습니다. 혹 해당 파일을 github에 업로드 하지 않았다면 jenkins 서버에 gradle을 설치하고,  `gradle wrap` 단계를 추가해 주어야 합니다. 

### 배포할 포트 확인

```bash
if ssh -o StrictHostKeyChecking=no ubuntu@\$nginx_ip "test -e /etc/nginx/sites-enabled/dev_8001"
then
    deployment_target_port=\$service_green_port
else
    deployment_target_port=\$service_blue_port
fi
```

현재 nginx가 바라보고 있지 않은 포트에 신규 프로세스를 배포하기 위해 포트를 확인합니다. 

### 배포할 포트로 실행중인 프로세스 중지

```bash

ssh -o StrictHostKeyChecking=no ubuntu@${explorer_service_server} "fuser -s -k \$deployment_target_port/tcp"  # 해당 port로 실행중인 프로세스 중지
for retry_count in \$(seq 10)          
    do
        if ssh ubuntu@${explorer_service_server} "test -z \$(fuser \$deployment_target_port/tcp)"; then
            echo "Server down successfully."
            sleep 2
            break
        fi

        echo "Retry After 5 secs"
        sleep 5
    done
```

여기서 해당 포트에 실행중인 프로세스를 확실하게 종료하지 않으면 포트 중복으로 인한 어플리케이션 오류가 발생합니다.

```bash
***************************
APPLICATION FAILED TO START
***************************

Description:

Web server failed to start. Port 8000 was already in use.

Action:

Identify and stop the process that's listening on port 8000 or configure this application to listen on another port.
```

해당 포트에 어플리케이션이 완전히 종료되기 전에 새로운 프로세스가 실행되는것을 방지하고자 2초의 여유를 두었습니다.

### 신규 jar 파일 서버로 복제 & 실행

```bash
# 신규 jar 파일을 app 서버로 복제
scp -o StrictHostKeyChecking=no ${JAR_FILE_PATH}/github-project-name-${VERSION}.jar ubuntu@${HOST}:${DEPLOY_PATH}
# app 서버에서 신규 jars 파일을 이용하여 프로세스 시작  
ssh -o StrictHostKeyChecking=no ubuntu@${HOSt} "nohup java -jar -Dserver.port=\$deployment_target_port -Dspring.profiles.active=dev -Xms4096m -Xmx8192m ${DEPLOY_PATH}/github-project${VERSION}.jar > ${DEPLOY_PATH}/${BUILD_NUMBER}.log &" &
```

jar 파일을 배포할 서버로 복제하고 프로세스를 실행합니다. 

### 신규 프로세스 정상 실행 확인

```bash
for retry_count in \$(seq 5)
	do
	  if curl -s "http://${HOST}:\$deployment_target_port" > /dev/null
	  then
	      echo "Health Check Successful"
	      break
	  fi
	
	  if [ \$retry_count -eq 10 ]
	  then
	    echo "Health Check Failed"
	    exit 1
	  fi
	
	  echo "Retry After 10 Secs"
	  sleep 10
	done
```

10초 주기로 실행한 프로세스가 해당 포트에서 정상적으로 동작하는지 확인합니다.

### reverse proxy 설정

```bash
if [ \$deployment_target_port == \$service_blue_port ]
	then
	    ssh -o StrictHostKeyChecking=no ubuntu@${nginx_ip} "sudo unlink /etc/nginx/sites-enabled/${ENV}_8000 && sudo ln -s /etc/nginx/sites-available/${ENV}_8001 /etc/nginx/sites-enabled && sudo systemctl reload nginx"
	else
	    ssh -o StrictHostKeyChecking=no ubuntu@${nginx_ip} "sudo unlink /etc/nginx/sites-enabled/${ENV}_8001 && sudo ln -s /etc/nginx/sites-available/${ENV}_8000 /etc/nginx/sites-enabled && sudo systemctl reload nginx"
fi
echo "Switch the reverse proxy direction of nginx to ${HOST}:\$deployment_target_port"
```

Health Check에 성공적으로 통과한 경우, nginx 서버에서는 reverse proxy 대상 포트를 새로운 앱 포트로 변경하고, 변경 사항을 적용하기 위해 리로드를 실행합니다. 이때, 이전에 사용하던 Active 상태였던 포트는 Backup 포트로 사용됩니다. 이러한 설정을 위해 nginx 서버의 `/etc/nginx/site-available` 디렉토리 아래에 두 개의 설정 파일을 생성하여 프록시 변경 시 해당 포트의 설정 파일을 사용하도록 구성하였습니다.

- Nginx 설정
    
    ```bash
    upstream backend {
        server {HOST}:8000 fail_timeout=60s max_fails=5;
        server {HOST}:8001 backup;
    }
    ...
    ```
    

프록시 방향이 잘 변경 되었다면 아래의 명령어로 확인 하였을때 해당 포트로 질의가 들어오는지 확인할 수 있습니다.

```groovy
sudo tcpdump -i bond0 src {nginx proxy server ip}
```

# 🫣 배포 결과

![스크린샷 2023-07-30 오후 10.42.44.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/10-stage-view.png)

각 스테이지가 정상적으로 동작하고 배포가 정상적으로 이루어진 모습입니다.

![스크린샷 2023-07-30 오후 10.47.30.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/11-console-log.png)

해당 배포의 로그를 확인해보면 8000포트로 배포된 것을 확인할 수 있는데요, 정상적으로 8000포트를 바라보고있는지 서버에서 명령어로 확인 해 보겠습니다.

![스크린샷 2023-07-30 오후 10.46.52.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/12-tcpdump.png)

8000번 포트로의 배포가 성공적으로 이루어졌습니다.

![스크린샷 2023-07-30 오후 10.55.13.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/13-grep.png)

`ps -ef | grep java` 명령어를 사용하여 8001번 포트도 Backup용으로 잘 남아있는점도 확인할 수 있습니다.

# 🙇🏻‍♀️ 마치며

nginx와 jenkins를 활용하여 CICD 구성을 완료했습니다. 해당 구성으로 로컬에서부터 배포서버까지의 단계를 버튼 클릭 한번으로 또는 github webhook을 통해 자동화 하여 개발과 배포과정을 크게 효율화하였습니다. 이러한 자동화는 서비스가 살아있는 상태에서 배포가 이루어지므로 우리 백엔드 개발자 뿐 아니라  프론트엔드 개발자분들 또한 안정적인 환경에서 작업할 수 있게 되었습니다.