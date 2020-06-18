---
title: "마이크로서비스 + DDD 간단하게 정리"
excerpt_separator: "<!--more-->"
categories:
  - develop
tags:
  - microservice
  - DDD
---
추후에 가볍게 마이크로서비스를 구현할 때 참고할 수 있도록 내용을 간단히 정리했습니다.
틀린 부분이 있으면 말씀주시면 감사드리겠습니다.

1. 하나의 마이크로서비스는 DDD 원칙에 따른 하나의 Aggregation(집합)에 따라 만들어야함.

2. 각 aggregation에서는 비즈니스 로직의 주체가되는 엔티티인 root 엔티티가 있음.

3. RDBMS 등을 이용할 때 Repository를 사용할 때, **웬만해서는** root 엔티티의 저장만을 위한 Repository만을 제공한다.
  - aggregation의 나머지 엔티티(또는 ValueObject)는 root 엔티티를 통해 수정될 수 있도록 한다.

4. 마이크로서비스간 통신은 곧 Aggregation root 간 참조와 같음(예: User와 Order간의 관계).

5. 한 Aggreagation에서 Aggregation root 엔티티를 참조할 때는 객체가 아닌 식별자(ID)를 사용할 것.

6. 마이크로서비스끼리 통신 종류: 동기 통신, 비동기 통신
  - 한 마이크로서비스에서 다른 마이크로서비스를 호출한 후, 결과값을 받아야할 때는 동기 통신
  - 다른 마이크로서비스 호출 후 결과를 알 필요 없이 이벤트 전달의 역할을 할 때는 비동기 통신
  - **비동기 IO와는 다르다.** 동기 통신이여도 비동기로 값을 받을 수 있음(async, await)
