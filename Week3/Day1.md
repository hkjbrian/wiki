# 2025-07-21

---

# 오늘의 일정 정리

--- 

- 10:00 ~ 10:30 데일리 스크럼
- 10:30 ~ 12:30 2주차 3단계 미션 코드 리팩터링
- 12:30 ~ 13:30 점심
- 13:00 ~ 18:30 3주차 1단계 미션 진행
- 18:30 ~ 19:00 주간 회고

# 배운 것들

---

### 작업 진행 중 위화감이 든다면 질문을 해보자

미션 3주차 1단계 미션의 요구사항 중 경로에서 지원하지 않는 메서드로 요청을 했을 경우 405 요청을 반환하도록 하는 부분을 구현하고 싶었다. 현재 내 구조에서는 405가 아닌 404 에러를 반환하게 되어서 이를 해결하고자 정적 파일 리졸버 자체를 제거하고, 각 경로마다 정적 파일을 처리하는 메서드를 추가하는 작업을 진행했다.  
작업을 진행하면 할수록 중복되는 코드가 늘어나 좋지 않은 방향으로의 리팩토링이라고 생각이 들었다. 5시쯤 시작한 작업을 1시간이 넘게 진행한 후 일간 회고를 진행하며 이 내용을 공유했더니 그룹원들이 좋지 않은 방법 같다고 다른 방법을 제시해주었다. 그제서야 내가 지나치게 무식한 방법을 사용하고 있었음을 깨닫게 되었다. 스스로도 맞는 방향이 아닌 것 같다고 자꾸 의심이 들 때는 질문을 해보도록 하자.

# 어려웠던 것들

--- 

### 예외 처리의 위치는 어디여야 할까?

회원 가입 과정에서의 검증 과정에 대한 예외 처리( 비어있는 폼, 중복 회원 가입 )등을 진행하면서 HttpRequest 를 바탕으로 User 객체를 생성하는 과정에서 어느 곳에서 예외를 터트려야 할지 의문이 생겼다.
1. 비어있는 값이 존재한다면, User 객체를 일단 생성하고, 비어있는 필드가 있다면 예외 처리
2. User 객체에 값을 넣기 전에 검증하여 예외 처리  

유사하게 메서드를 호출한 곳에서 예외 처리를 해야할지, 메서드의 내부에서 예외 처리를 해야할지에 대해 모호했던 경험도 많았던 것 같다.

항상 유효해야하는 도메인 객체의 경우는 내부적으로 생성 과정에서 예외 처리가 필요하고, 입력 값을 모두 받은 후 한 번에 검증하고 싶다면 Validator 를 만들어서 예외를 던지도록 해도 된다고는 하는데  
깔끔히 떨어지지 않는 답 같아서 그룹 세션 시간에 여러 의견을 들어보면 좋을 것 같다.

# 일간 회고

--- 

## Keep

### 문제 정의하기

문제를 정의하는 과정에서 내가 해야 할 일이 더욱 명확해짐을 느끼는 것 같다.  
단계별로 문제를 나누고, 현재 상황에서 문제를 해결하기 위해 필요한 것들을 정리하는 과정이 작업을 진행하는데에 있어서 정말 큰 도움이 되는 것 같다. 계속 유지하도록 하자

## Problem

### 한 곳에서 내가 해야 할 작업들을 관리하자

현재는 README.md 를 통해서 내가 해야 할 일들의 목록을 정리해두고 있는데, 그룹 세션 혹은 스쿼드 세션에서 받은 피드백의 경우 README 에 작성하지 않아서 나중에 까먹게 되는 문제가 있다.  
모든 작업과 관련한 내용을 한 곳에서 관리하도록 하자

## Try

### 테스트 작성하기

1,2주차에는 구조를 구상하고 단계별로 리팩토링 할 내용이 많아 테스트 작성을 할 기회가 없었는데, 3주차의 경우에는 구조가 바뀌지 않아도 되므로 테스트를 작성해보도록 하자.

--- 


