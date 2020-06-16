---
title: "ECS Fargate에 Node.js Docker 컨테이너 배포 (1) - ECS 서비스 생성 및 Docker 컨테이너 실행"
excerpt_separator: "<!--more-->"
categories:
  - develop
tags:
  - backend
  - nodejs
  - docker
  - aws
  - ecs
---

최근 핫한 AWS, Docker를 공부하면서 AWS에 Dockerize한 앱을 배포하는 방법을 정리할 겸 포스팅을 하게되었습니다.

---
읽기 전 사전 준비 사항:
- git 기초지식
- Node.js 설치
- aws-cli 설치 **(python3 이상에서 설치 가능합니다)**
- Docker Desktop 설치
- AWS 계정 생성

# 프로젝트 생성 & Docker app 작성

우선 프로젝트를 생성해보겠습니다.

~~~
$ md my-app
$ cd my-app
$ npm init -y
$ git init
~~~

그리고 `npm install --save express` 명령어로 express 모듈을 설치해줍니다. 설치후에 index.js 파일을 생성하여 간단한 API 서버 프로그램을 작성합니다.

**index.js**
~~~
const app = require("express")();

app.get("/", (req, res) => {
  res.json({
    message: "Success",
  });
});

app.get("/hello", (req, res) => {
  res.json({
    message: "hello",
  });
});

app.listen(3000, () => {
  console.log("Server starting on port 80");
});

~~~

`node index.js` 명령어를 입력하여 실행이 되는지 확인해주세요.
http://localhost:3000 링크를 통해 확인 가능합니다.  
확인이 끝났으면 실행한 프로세스를 종료한 뒤, Dockerfile을 작성합니다.

**Dockerfile**
~~~
FROM node:10.13-alpine # 1

WORKDIR /home/app # 2
COPY . . # 3

RUN npm install # 4
CMD node index.js # 5
~~~

node_modules 디렉토리는 아무래도 Dockerfile에서 COPY를 통해 복사되기 보다는 RUN npm install 명령어를 통해 생성되는게 깔끔하겠죠? node_modules 디렉토리가 복사되지 않도록 .dockerignore 파일을 작성해줍니다.  
__(.gitignore 파일도 아래와 같은 내용으로 생성해주세요!)__

**.dockerignore**
~~~
node_modules/
~~~

.gitignore, .dockerignore 파일 모두 작성 완료했으면 이제 Dockerfile을 빌드하여 docker image를 생성해봅시다.
`docker build -t my-app` 명령어로 my-app이라는 이름의 docker image를 생성합니다.
~~~
$ docker build -t my-app .

Sending build context to Docker daemon  83.46kB
Step 1/5 : FROM node:10.13-alpine
 ---> 93f2dcbcddfe
Step 2/5 : WORKDIR /home/app
 ---> Using cache
 ---> 458c745b5734
Step 3/5 : COPY . .
 ---> 1a163c57e9ab
Step 4/5 : RUN npm install
 ---> Running in aa294de2b001
npm WARN my-app@1.0.0 No description
npm WARN my-app@1.0.0 No repository field.

added 50 packages from 37 contributors and audited 50 packages in 1.722s
found 0 vulnerabilities

Removing intermediate container aa294de2b001
 ---> ff29771ecafd
Step 5/5 : CMD node index.js
 ---> Running in 5ff3dce0e92f
Removing intermediate container 5ff3dce0e92f
 ---> 3b8cb1a7fdd9
Successfully built 3b8cb1a7fdd9
Successfully tagged my-app:latest
~~~
위와 같은 결과가 나오면 성공입니다.

이제 `docker run` 명령어를 통해 생성한 my-app을 container로 실행해봅시다.
~~~
$ docker run -d -p 3000:3000 --name my-app-container my-app
~~~

성공적으로 container를 실행시키면, 콘솔에 컨테이너 해시값이 출력될 것입니다.  
이제 `docker ps` 명령어로 my-app-container 라는 이름의 인스턴스가 존재하는지 확인해주세요.
~~~
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
1d7e38723a58        my-app              "/bin/sh -c 'node in…"   9 minutes ago       Up 9 minutes        0.0.0.0:3000->3000/tcp   my-app-container
~~~

ps를 통해 container가 실행중인걸 확인했습니다. 그럼 이번엔 정말로 기능이 동작하는지 확인해봅시다.  
http://localhost:3000 나 http://localhost:3000/hello 로 접속하여 확인해주세요.
![](/assets/img/2020-05-22_01.png)

실행한 컨테이너는 `docker stop` 명령어로 종료시켜줍니다.
 ~~~
$ docker stop my-app-container
my-app-container
 ~~~

# 생성한 Docker Image를 저장하려면? Amazon ECR에 PUSH

ECS에 생성한 docker image를 서비스로 배포하려면 여러 사전 작업들이 필요합니다. ECS에 배포하기전, docker image를 먼저 ECR에 푸시해봅시다.

웹페이지를 열고 AWS 콘솔 서비스에서 IAM 서비스 창으로 들어가줍니다.  
"사용자 추가" 버튼을 클릭한 뒤 사용자를 추가합니다.  
![](/assets/img/2020-05-22_03.png)

"다음" 버튼을 누른 후에 "기존 정책에 직접 연결", EC2ContainerRegistryFullAccess 권한을 선택한 후 "다음" 버튼을 눌러주세요.
![](/assets/img/2020-05-22_07.png)

최종적으로 아래와 같은 권한을 가진 사용자를 생성하시면 됩니다.
![](/assets/img/2020-05-22_04.png)
"사용자 만들기" 버튼을 눌러 다음 화면으로 넘어가시면, 이 계정에 액세스할 수 있는 AWS Access Key ID와 AWS Secret Access Key 정보를 볼 수 있습니다.  
.csv 파일을 다운받거나 메모를 하여 기록해주세요.

Amazon ECR 페이지로 들어가서 리포지토리를 생성해주세요.
![](/assets/img/2020-05-22_02.png)

이제 리포지토리를 생성했으니, 이 리포지토리에 우리가 만든 이미지를 푸시하면 되겠습니다.

`aws configure` 명령어로 로컬에 aws-cli 계정 정보를 설정합니다. 아까 저장한 사용자 계정의 Access Key Id, Secret Access Key를 아래를 참고하여 입력하면 되겠습니다.
~~~
$ aws configure
AWS Access Key ID [****************4Y5H]: <저장한 Access Key Id>
AWS Secret Access Key [****************wxB8]: <저장한 Secret Access Key>
Default region name [ap-northeast-2]: ap-northeast-2
Default output format [json]: json
~~~

그 다음에 생성한 리포지토리로 들어가서 '푸시명령보기' 버튼을 클릭합니다.
![](/assets/img/2020-05-22_08.png)
![](/assets/img/2020-05-22_09.png)

각 과정대로 명렁어를 입력하시면 정상적으로 푸시가 완료됩니다.

ECR의 my-app 리포지토리로 들어가서 잘 푸시됬는지 확인해봅시다.
![](/assets/img/2020-05-22_06.png)
잘 된것 같네요 :)

# 서비스의 네트워크 환경을 구성하는 VPC 생성하기 & 클러스터 생성하기

#### *(이 부분은 클러스터 생성 시에 한 번에 생성해줄 수 있는 버튼이 있으니, 노가다를 싫어하시면 바로 클러스터 생성 부분으로 넘어가주세요)*
ECS 컨테이너를 동작시킬 격리된 네트워크 환경을 구성하기 위해 VPC 설정을 해줍니다.

우선 VPC를 생성해주세요
![](/assets/img/2020-05-22_16.png)

VPC 생성을 완료했다면, VPC 내부의 서브넷 생성을 해야합니다.  
VPC 하나 당 서브넷 가용 영역이 2개 이상이여야 하니, my-app-subnet-a를 아래와 같이 생성하고, my-app-subnet-b도 생성해야합니다. vpc는 위에서 생성한 my-app-vpc와 연결해주세요.
![](/assets/img/2020-05-22_17.png)
my-app-subnet-b의 경우는 가용 영역은 위 그림과 다른 것으로 설정하고, IPV4 CIDR 블록은 10.0.2.0/24 로 설정하면 됩니다.

두 서브넷 모두 생성을 완료하면 다음과 같이 확인할 수 있습니다.
![](/assets/img/2020-05-22_18.png)

서브넷으로부터 인터넷을 통한 통신을 위해 인터넷 게이트웨이를 생성합니다.
![](/assets/img/2020-05-22_19.png)

그 다음, 생성된 인터넷 게이트웨이를 my-app-vpc와 연결합니다.
![](/assets/img/2020-05-22_21.png)
![](/assets/img/2020-05-22_22.png)

인터넷 게이트웨이를 VPC와 연결하셨다면, 이제 VPC의 라우팅 테이블의 라우트를 해야합니다.
라우팅 테이블에 인터넷 게이트웨이를 추가하기 전에는, VPC 내부에서만 통신할 수 있도록 local 라우트가 기본적으로 설정되어있음을 확인할 수 있습니다.
각 서브넷이 인터넷에 연결할 수 있도록 라우팅 테이블을 편집해야합니다.  
**라우팅 편집** 버튼을 눌러 라우팅 편집 페이지로 이동한 다음, 라우팅을 추가한 뒤 아래와 같이 설정해주세요.
(local 제외 모든 라우트를 인터넷 게이트웨이와 연결)
![](/assets/img/2020-05-22_20.png)
![](/assets/img/2020-05-22_23.png)

VPC 구성을 끝마쳤습니다.  
이제 클러스터를 생성해봅시다.

![](/assets/img/2020-05-22_15.png)
![](/assets/img/2020-05-22_24.png)

간단하게 생성가능합니다.

# 컨테이너들을 어떻게 실행할까? Task Definition 생성

ECS는 크게 Task definition, Cluster, Service로 구성되어 있습니다.
Task definition은 ECR 리포지토리 등으로 푸시한 Docker Image를 컨테이너로 실행할 때, 어떻게 실행할 것인지 정의합니다.

ECS > 작업 정의 페이지에서 **새 작업 정의 생성 버튼**을 클릭해주세요.
 
Fargate는 EC2와 달리 컨테이너를 실행한 시간만큼 지불한다는 특징이 있습니다. EC2 인스턴스의 경우 실행하지 않는 시간에도 인스턴스가 켜져있기 때문에 요금을 지불합니다. 많이 실행될 일 없는 간단한 작업일 경우 Fargate가 효율적입니다.

시작 유형 호환성 선택에서 **Fargate**를 선택하시고 다음 단계를 눌러주세요.
![](/assets/img/2020-05-22_10.png)

아래와 같이 설정합니다.
![](/assets/img/2020-05-22_11.png)
![](/assets/img/2020-05-22_12.png)

컨테이너 정의 부분에서 추가 버튼을 누르면 아래와 같은 창이 뜨는데, 나머지 설정 없이 여기 보이는 부분만 설정하시면 됩니다. 아래 이미지 부분의 이미지 주소는
![](/assets/img/2020-05-22_14.png)

ECR 리포지토리의 my-app 으로 들어가서 latest 태그 이미지의 url을 복사하시면 됩니다.
![](/assets/img/2020-05-22_13.png)

# HTTP 요청을 Docker app으로 전송하는 로드밸런서 생성하기

EC2 > 로드밸런서 페이지로 가서 로드 밸런서 생성 버튼을 클릭하세요.
**Application Load Balancer** 를 생성합니다.
![](/assets/img/2020-05-22_25.png)

로드밸런서 이름 포트 등을 설정하는 모습입니다.
![](/assets/img/2020-05-22_26.png)

가용 영역부분은 생성한 VPC에 속해있는 my-app-subnet-a, my-app-subnet-b를 선택해주세요.
(가용영역별로 서브넷을 나눈 이유는 로드밸런서 생성 시 가용영역이 두 개 이상 지정되어야 하기 때문입니다).
![](/assets/img/2020-05-22_27.png)

HTTP 웹 서버를 동작시킬것이므로 80번 포트를 열어두세요.
![](/assets/img/2020-05-22_28.png)

대상 그룹을 생성합니다. application load balancer의 경우 트래픽을 대상 그룹내에 속해있는 EC2 인스턴스 또는 ip 또는 lambda 함수로 라우팅합니다.  
**Fargate는 EC2 인스턴스 위에서 동작하지 않으므로 ip**를 선택합니다.
![](/assets/img/2020-05-22_29.png)

검토 후에 생성한 뒤 로드밸런서와 대상 그룹이 제대로 생성되었는지 확인할 수 있습니다.
![](/assets/img/2020-05-22_31.png)
![](/assets/img/2020-05-22_30.png)

# 작업 정의를 토데로 컨테이너를 관리하는 ECS 서비스 만들기

ECS > 클러스터 페이지에서 하단의 서비스 항목의 생성 버튼을 눌러주세요.
![](/assets/img/2020-05-22_32.png)

서비스 구성은 아래와 같이 구성합니다.  
시작 유형은 Fargate로 설정하고, 이전에 정의한 my-app-task를 작업 정의 패밀리로 설정합니다.  
서비스 이름을 지어준 뒤에 작업 개수는 1로 설정합니다.  
배포 관련 설정은 기본적으로 설정되어 있는 롤링 업데이트를 사용합니다.
![](/assets/img/2020-05-22_33.png)

다음으로 넘어가서, 클러스터 VPC는 이전에 생성한 my-app-vpc로 설정합니다.  
서브넷도 마찬가지로 my-app-subnet-a, my-app-subnet-b로 설정해주세요.  
보안 그룹은 아래를 참고하여 새로 생성합니다.
![](/assets/img/2020-05-22_34.png)
3000 포트를 열어주는 이유는 작성한 node.js 서버가 port 3000번으로 요청을 받도록 설정했기 때문입니다.

VPC 및 보안그룹 설정이 끝나면 아래와 같이 설정되었음을 확인해보세요.
![](/assets/img/2020-05-22_35.png)

아래는 로드밸런싱 관련 설정입니다. 위에서 생성한 my-app-elb 로드밸런서를 선택하고, 대상 그룹 이름 또한 위에서 생성한 my-app-target-group으로 생성하면 끝입니다! 다음으로 넘어가주세요.
![](/assets/img/2020-05-22_36.png)

Auto scaling은 따로 설정해주지 않습니다.
![](/assets/img/2020-05-22_37.png)

여기까지 완료되면 서비스가 생성되고 작성한 작업 정의대로 작업이 실행되어 컨테이너가 실행됨을 확인할 수 있습니다.
![](/assets/img/2020-05-22_38.png)

이제 한 번 제대로 배포가 되었는지 확인해봅시다.

EC2 > 로드밸런서 페이지로 들어간 뒤, DNS 이름을 복사하여 복사한 url로 이동해 봅시다.
![](/assets/img/2020-05-22_40.png)

성공적으로 응답하는 것을 볼 수 있습니다!
![](/assets/img/2020-05-22_41.png)

다음 글에서는 github와 연동하여 push시에 자동으로 배포가 진행되도록 하는 과정을 담아보겠습니다.

