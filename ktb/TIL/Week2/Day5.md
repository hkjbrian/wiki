# 2025-09-19

---

# 오늘의 일정 정리

--- 

- 09:00 ~ 10:00 데일리 스크럼 진행
- 10:00 ~ 14:00 딥다이브 진행 ( synchronized 와 volatile 에 대한 이해 ) 
- 14:00 ~ 15:00 스레드의 생명주기 이해
- 15:00 ~ 17:30 딥다이브 내용 공유
- 17:30 ~ 18:00 수업 평가 및 마무리

# 느낀 점

---

### 최고의 공부 방법은 설명하는 것

오늘 synchronized 에 대해서 딥다이브하는 시간을 가졌다. Lock 의 개념과 자주 비교되기 때문에 특별히 신경써서 내용을 정리했던 것 같다. 
발표 내용은 synchronized 와 volatile 의 차이점이었는데 팀원들이 synchronized 의 동작 원리 이해에 어려움을 겪고 있어서 따로 약 30분간 이야기를 나누었다.  
이 과정에서 다른 사람들이 이해하기 어려워하는 내용을 단계별로 설명해가면서 나 스스로도 더욱 이해가 견고해지는 것을 느꼈다. 팀원들의 고마움 표시로 인한 뿌듯함도 덤으로 얻을 수 있었다.  
이 내용을 코드를 추가해 배움 일기로도 정리하여 올려두었는데 가장 최고의 공부 방법이라고 느껴져서 앞으로의 학습 방식에 잘 적용해보면 좋을 것 같다

# 오늘 학습한 내용
synchronized 와 volatile 에 관해 깊게 고민해 부분은 다음과 같다
- synchronized 가 획득하는 락은 뭘까?

- synchronized 에서 락 획득에 실패하여 대기하는 스레드들은 어디에서 관리할까?

- volatile은 어떻게 동작할까?


[synchronized 와 volatile](https://tidal-tub-cac.notion.site/synchronized-volatile-273e569146a68047ac63eed1b1b20431?source=copy_link)  
[synchronized의 이해](https://tidal-tub-cac.notion.site/synchronized-271e569146a68060adc3f3f4c73faaba?source=copy_link)

---

