---
title: "ECS Fargate에 Node.js Docker 앱 배포 - (1) Docker Image를 ECR에 업로드하기"
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
Node.js로 아주 간단하게 API 서버를 만들고 배포하는 과정을 담았습니다.
질문/지적 환영합니다.

---
읽기 전 사전 준비 사항:
- git 기초지식
- Node.js 설치
- python 3 이상 버전 설치 (또는 aws-cli 설치)
- Docker Desktop 설치
- AWS 계정 생성

# 프로젝트 생성

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

app.listen(80, () => {
  console.log("Server starting on port 80");
});

~~~

`node index.js` 명령어를 입력하여 실행이 되는지 확인해주세요.
http://localhost 링크를 통해 확인 가능합니다.  
확인이 끝났으면 실행한 프로세스를 종료한 뒤, Dockerfile을 작성합니다.

**Dockerfile**
~~~
FROM node:10.13-alpine # 1

WORKDIR /home/app # 2
COPY . . # 3

RUN npm install # 4
CMD node index.js # 5
~~~

- #1 사용할 node.js 버전을 표기한 Docker image를 기반으로 합니다.
- #2 Docker container instance 내부의 /home/app 경로를 작업 경로로 설정합니다.
- #3 현재 디렉토리에 있는 파일들을 Dokcer instance 내부 작업 경로 디렉토리(/home/app)에 모두 복사합니다.
- #4 npm install로 package.json에 명시된 모듈들을 설치합니다. 이 프로젝트이 경우 express 관련 모듈이 설치됩니다.
- #5 인스턴스 시작시에 node index.js 명령어가 시작되도록 설정해줍니다.

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
$ docker run -d -p 80:80 --name my-app-container my-app
~~~
- -d: demon 프로세스(background)로 실행 
- -p: 호스트의 포트와 docker container 내부 포트를 연결해줍니다. (<호스트 포트>:<컨테이너 포트>)
- --name: 현재 실행될 컨테이너의 이름을 지정합니다.

성공적으로 container를 실행시키면, 콘솔에 컨테이너 해시값이 출력될 것입니다.  
이제 `docker ps` 명령어로 my-app-container 라는 이름의 인스턴스가 존재하는지 확인해주세요.

~~~
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
1d7e38723a58        my-app              "/bin/sh -c 'node in…"   9 minutes ago       Up 9 minutes        0.0.0.0:80->80/tcp   my-app-container
~~~

ps를 통해 container가 실행중인걸 확인했습니다.  
그럼 이번엔 정말로 기능이 동작하는지 확인해봅시다.

http://localhost 나 http://localhost/hello 로 접속하여 확인해주세요.

![](/assets/img/2020-05-22_01.png)

실행한 컨테이너는 `docker stop` 명령어로 종료시켜줍니다.
 ~~~
$ docker stop my-app-container
my-app-container
 ~~~

# Amazon ECR에 생성한 이미지 Push

ECS에 생성한 docker image를 서비스로 배포하려면 여러 사전 작업들이 필요합니다. ECS에 배포하기전, docker image를 먼저 ECR에 푸시해봅시다.

우선 aws-cli를 설치해야합니다. Mac os 기준으로 python3에서 제대로 작동합니다.

설치가 끝났으면, 웹페이지를 열고 AWS 콘솔 서비스에서 IAM 서비스 창으로 들어가줍니다.  
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

AWS ECR에 푸시할 권한이 필요하므로 docker 로그인을 먼저 해야합니다.
docker 로그인을 위해서는... ECR 접근 권한을 통해 docker login 정보를 얻어야합니다. 참 깁니다.

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