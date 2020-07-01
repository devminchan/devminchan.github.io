---
title: "ECS Fargate에 Docker container 배포 (3) - 환경변수 안전하게 넘겨주기"
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

이번 글에서는 배포된 서비스의 컨테이너에 DB password 등 민감한 정보를 가진 환경변수를 안전하게 넘기는 법을 알려드리겠습니다.

# 환경변수 응답 코드 작성
우선 app.js의 /test 경로 핸들러를 아래와 같이 수정해주세요.  
(컨테이너에 환경변수가 들어갔는지 확인하기 위함)

**app.js**
```js
app.get("/test", (req, res) => {
  res.json({
    message: "Test",
    env: process.env.TEST_ENV || "NULL",
  });
});
```

이제 git에 커밋후 원격 저장소에 푸시해주세요.  
지난 글에서 github action으로 자동 배포를 하도록 설정했기 때문에 바로 결과를 확인해볼 수 있습니다.

my-app 레포지토리의 Actions 항목에서 배포 workflow가 작동하는지 확인해볼수 있습니다.
![](/assets/img/2020-06-30/01.png)

배포가 완료되면 로드밸런서 DNS를 url에 넣고 /test 경로로 접속하여 받은 응답을 확인해주세요.
![](/assets/img/2020-06-30/02.png)

현재는 환경변수가 설정되어 있지 않아서 env값이 NULL 인 것을 확인할 수 있습니다.

# SecretsManager에 환경변수 설정하기
이제 환경변수를 ECS로 배포된 컨테이너에 한 번 주입해봅시다.  
AWS 콘솔의 Secrets Manager 콘솔로 들어가서 새 보안 암호 저장 버튼을 클릭해주세요.
![](/assets/img/2020-06-30/03.png)

아래 화면을 참고해서 보안암호에 들어갈 값을 입력해주면 됩니다.  
그런뒤에 다음 버튼을 눌러주세요.
![](/assets/img/2020-06-30/04.png)

다른 프로젝트/서비스에서 사용할 환경변수와 겹치지 않도록 <프로젝트명>/<환경변수명>으로 이름을 설정해줍시다. 설정이 완료되면 다음 버튼을 눌러주세요.
![](/assets/img/2020-06-30/05.png)

아래와 같이 보안암호가 생성된 것을 확인할 수 있습니다.
![](/assets/img/2020-06-30/06.png)

생성한 보안암호 myApp/TEST_ENV의 보안암호 ARN을 복사해주세요
![](/assets/img/2020-06-30/07.png)

그리고 task-definition의 secrets 값에 배열을 추가하시고 배열의 값에 TEST_ENV 환경변수 정보를 담고있는 json을 추가해줍시다.  
json의 valueFrom 값에는 복사한 ARN 값을 넣어주시면 됩니다.  

**task-definition.json**
```json
.
.
.
"secrets": [
        {
          "name": "TEST_ENV",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:129403376983:secret:myApp/TEST_ENV-oZpwxw"
        }
      ],
.
.
.
```

이제 푸시해서 결과를 확인해봅시다.
```bash
$ git add .
$ git commit -m "Secrets 추가"
$ git push -u origin master
```

배포 프로세스 도중 실패했는데요, task-definition의 작업 실행 IAM 역할이 AWS Secrets Manager의 보안암호에 접근할 권한이 없어서 발생한 오류입니다.
![](/assets/img/2020-06-30/09.png)

# SecretsManager에 접근할 IAM 역할 생성하기
이를 해결하기 위해서 작업 실행 IAM 역할을 새로 생성합니다.
IAM > 역할 페이지에서 역할 만들기 버튼을 눌러주세요.
![](/assets/img/2020-06-30/11.png)

사용 사례는 __Elastic Container Service > Elastic Container Service Task__ 를 선택해주세요(크게 중요하지는 않아보입니다).  
연결할 정책은 **AmazonECSTaskExecutionPolicy** 입니다.
![](/assets/img/2020-06-30/12.png)

역할 이름은 **ecsTaskExecutionWithSecretsRole**로 설정했습니다.
![](/assets/img/2020-06-30/13.png)

역할이 생성되었으면 이제 Secrets Manager의 보안암호에 접근할 정책을 새로 만들어줍시다.  
**인라인 정책 추가** 버튼을 클릭해주세요.
![](/assets/img/2020-06-30/14.png)

아래 사진을 참고하여 Secrets Manager 관련 작업 정책을 생성합니다.
![](/assets/img/2020-06-30/15.png)
![](/assets/img/2020-06-30/16.png)
![](/assets/img/2020-06-30/17.png)

정책 생성이 완료되었으면 위에서 생성한 역할에 SecretsManagerPolicy가 적용된 것을 확인할 수 있습니다.  
역할 ARN을 복사해주세요!
![](/assets/img/2020-06-30/18.png)

이제 프로젝트의 task-definition.json의 executionRoleArn을 복사한 IAM 역할 ARN으로 수정해주세요.

**task-definition.json**
```json
{
  .
  .
  .
  "executionRoleArn": "arn:aws:iam::129403376983:role/ecsTaskExecutionWithSecretsRole",
  .
  .
  .
}
```

그런 다음 수정 사항을 커밋한 후 푸시를 진행합니다.  

성공적으로 배포되었네요.
![](/assets/img/2020-06-30/19.png)

테스트를 한 번 해보겠습니다. <로드밸런서 DNS>/test 경로로 접속할 시
![](/assets/img/2020-06-30/20.png)

SecretsManager에서 설정한 myApp/TEST_ENV의 값이 출력되는 것을 확인할 수 있습니다.


