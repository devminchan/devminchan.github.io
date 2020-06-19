---
title: "ECS Fargate에 Docker container 배포 (2) - github action CI/CD 구현"
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

이번 글에서는 저번에 올린 ecs 서비스의 container를
코드 변경 후 github에 push 시 자동으로 Docker Image를 빌드하고, 새로운 작업 정의(Task definition)을 생성 및 ecs에 배포하는 프로세스를 구축해보겠습니다.

# Task definition 파일 생성하기

ECS > 작업 정의 > my-app-task > my-app-task:1 으로 가셔서 JSON 보기 버튼을 눌러주세요.
![](/assets/img/2020-06-19/01.png)


나타나는 json을 모두 복사한 뒤, 프로젝트의 루트 경로에 task-definition.json 파일을 생성하고,
복사한 내용을 붙여주세요.

**task-definition.json**
```json
{
  "ipcMode": null,
  "executionRoleArn": "arn:aws:iam::129403376983:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "dnsSearchDomains": null,
      "environmentFiles": null,
      "logConfiguration": {
        "logDriver": "awslogs",
        "secretOptions": null,
        "options": {
          "awslogs-group": "/ecs/my-app-task",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "entryPoint": null,
      "portMappings": [
        {
          "hostPort": 3000,
          "protocol": "tcp",
          "containerPort": 3000
        }
      ],
      "command": null,
      "linuxParameters": null,
      "cpu": 0,
      "environment": [],
      "resourceRequirements": null,
      "ulimits": null,
      "dnsServers": null,
      "mountPoints": [],
      "workingDirectory": null,
      "secrets": null,
      "dockerSecurityOptions": null,
      "memory": null,
      "memoryReservation": null,
      "volumesFrom": [],
      "stopTimeout": null,
      "image": "129403376983.dkr.ecr.ap-northeast-2.amazonaws.com/my-app:latest",
      "startTimeout": null,
      "firelensConfiguration": null,
      "dependsOn": null,
      "disableNetworking": null,
      "interactive": null,
      "healthCheck": null,
      "essential": true,
      "links": null,
      "hostname": null,
      "extraHosts": null,
      "pseudoTerminal": null,
      "user": null,
      "readonlyRootFilesystem": null,
      "dockerLabels": null,
      "systemControls": null,
      "privileged": null,
      "name": "my-app-container"
    }
  ],
  "placementConstraints": [],
  "memory": "512",
  "taskRoleArn": null,
  "compatibilities": ["EC2", "FARGATE"],
  "taskDefinitionArn": "arn:aws:ecs:ap-northeast-2:129403376983:task-definition/my-app-task:1",
  "family": "my-app-task",
  "requiresAttributes": [
    {
      "targetId": null,
      "targetType": null,
      "value": null,
      "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
    },
    {
      "targetId": null,
      "targetType": null,
      "value": null,
      "name": "ecs.capability.execution-role-awslogs"
    },
    {
      "targetId": null,
      "targetType": null,
      "value": null,
      "name": "com.amazonaws.ecs.capability.ecr-auth"
    },
    {
      "targetId": null,
      "targetType": null,
      "value": null,
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.19"
    },
    {
      "targetId": null,
      "targetType": null,
      "value": null,
      "name": "ecs.capability.execution-role-ecr-pull"
    },
    {
      "targetId": null,
      "targetType": null,
      "value": null,
      "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
    },
    {
      "targetId": null,
      "targetType": null,
      "value": null,
      "name": "ecs.capability.task-eni"
    }
  ],
  "pidMode": null,
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "revision": 1,
  "status": "ACTIVE",
  "inferenceAccelerators": null,
  "proxyConfiguration": null,
  "volumes": []
}
```

# 기존 ECR 유저에 추가 권한 붙이기

IAM > 사용자 페이지에서 이전 포스팅에서 생성한 ecr-user-1에 ECS 배포를 위해 AmazonECS_FullAccess 권한을 추가해줍니다.
![](/assets/img/2020-06-19/02.png)
![](/assets/img/2020-06-19/03.png)

# github action 추가하기

github action을 통해 자동 배포(CD) 기능을 구축해봅시다.  
우선 프로젝트를 올릴 깃헙 레포지토리를 새로 만들어주세요.
![](/assets/img/2020-06-19/04.png)

그 다음 로컬 프로젝트 폴더에서 현재 추가된 파일들 모두 커밋한 다음에,
저장소로 푸시해줍시다.  
우선 커밋을 진행해주세요.

```sh
$ git add .
$ git commit -m "First commit"
```

커밋이 완료되었으면 푸시를 진행합니다.
```sh
$ git remote add origin https://github.com/devminchan/my-app.git
$ git push -u origin master
```

푸시가 완료되면, github action에서 빌드/배포할 때 ECS, ECR 등 AWS 서비스에 접근할 수 있도록 계정 정보를 설정해줍시다.  
Settings > Secrets 로 가서

**AWS_ACCESS_KEY_ID**  
**ASWS_SECRET_ACCESS_KEY**

각각의 Name을 위와같이 설정하여 secret에 추가해주세요.  
Value에는 각각 ecr-user-1의 accessKeyId, secretAccessKey를 넣으시면 됩니다.
![](/assets/img/2020-06-19/08.png)

두 시크릿을 모두 추가했다면 아래와 같이 보입니다.
![](/assets/img/2020-06-19/05.png)

이제 진짜 자동배포를 위한 작업을 시작해봅시다.  
Actions 페이지로 가서 Deploy to Amazon ECS workflow를 선택해주세요(Set up this workflow 클릭).
![](/assets/img/2020-06-19/06.png)

workflow가 적혀있는 aws.yml에서 몇 몇 수정해야할 부분이 있습니다.

1. on: release: ~ 부분을 on:push:branches: [master]로 변경해주세요. 마스터 브랜치에 push 할 때 마다 자동으로 docker image를 빌드 및 배포를 진행할 것이므로 변경해줍니다.
2. aws-region을 ap-northeast-2로 설정해주세요.
3. 환경변수 ECR_REPOSITORY의 값을 my-app으로 변경해주세요.
4. 환경변수 container-name의 값을 my-app-container로 설정해주세요.
4. 환경변수 service, cluster 의 값을 각각 my-app-ecs-service, my-app-container로 설정해주세요(Deploy Amazon ECS task definition).

최종적인 aws.yml 파일의 내용은 다음과 같습니다.
```yml
on:
  push:
    branches: [ master ]

name: Deploy to Amazon ECS

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-2

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build, tag, and push image to Amazon ECR
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: my-app
        IMAGE_TAG: ${{ github.sha }}
      run: |
        # Build a docker container and
        # push it to ECR so that it can
        # be deployed to ECS.
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

    - name: Fill in the new image ID in the Amazon ECS task definition
      id: task-def
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      with:
        task-definition: task-definition.json
        container-name: my-app-container
        image: ${{ steps.build-image.outputs.image }}

    - name: Deploy Amazon ECS task definition
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: my-app-ecs-service
        cluster: my-app-cluster
        wait-for-service-stability: true

```

설정 완료 후 Start commit > Commit new file 버튼을 클릭해서 master 브랜치에 커밋을 등록해주세요.
![](/assets/img/2020-06-19/07.png)

그런 다음 Actions 페이지의 workflows 목록을 확인하여 들어가보면, 배포 작업 진행 현황을 확인할 수 있습니다.  
정상적으로 성공시에 아래와 같은 화면을 확인할 수 있습니다.
![](/assets/img/2020-06-19/09.png)

이제 코드를 변경하여 수정 사항이 자동으로 적용되고 배포되는 모습을 확인해봅시다.
index.js에 GET 메서드로 /test 경로로 접속시 json을 응답하는 핸들러를 추가했습니다.

**index.js**
```js
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

// 테스트용
app.get("/test", (req, res) => {
  res.json({
    message: "Test",
  });
});

app.listen(3000, () => {
  console.log("Server starting on port 80");
});
```

변경 사항을 추가하고 커밋한 뒤, 푸시해주세요.
```sh
$ git add .

$ git commit -m "배포 테스트"

$ git push -u origin master
```

푸시가 완료되면, Actions 탭에서 배포가 잘되는지 한 번 지켜봅시다.
![](/assets/img/2020-06-19/10.png)

성공하면 아래와 같은 화면을 볼 수 있습니다.
![](/assets/img/2020-06-19/11.png)

이제 로드밸런서 dns의 test 경로로 접속하여 정말 적용이 됬는지 확인해봅시다.
aws 콘솔의 EC2 > 로드밸런서 페이지에서 my-app-elb의 dns이름을 복사하고 브라우저든 Postman이든 curl이든 아무거나 써서 /test 경로로 접속하여 추가한 핸들러가 제대로 동작하는지 확인해봅시다.

저는 이번에는 curl로 확인해보겠습니다.
```sh
$ curl -X GET my-app-elb-1003967422.ap-northeast-2.elb.amazonaws.com/test 

{"message":"Test"}
```
추가한 핸들러의 응답값이 돌아오네요. 성공입니다!

이렇게해서 ECS 서비스에 Fargate로 Docker app을 자동으로 배포하는 과정을 끝마쳤습니다.  
다음에는 ECS service에 deploy되는 Docker 컨테이너에 DB 정보든 민감한 환경변수를 전달하는 법을 포스팅하겠습니다.