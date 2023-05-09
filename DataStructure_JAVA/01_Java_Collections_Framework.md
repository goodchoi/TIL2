# 자바 컬렉션 프레임워크 (Java Collections Framework)

>  출처 : [Stranger's LAB (tistory.com)](https://st-lab.tistory.com/142?category=856997)



## 1. 의미

##### 컬렉션(Collection)이란

* 일정한 부류의 것을 수집하여 한 공간에 모아놓은 것.
  * ex) 동전수집, 우표수집..
  * 데이터를 쉽게 다루기 위해 지원하는 자료구조.

##### 프레임 워크(Framework)란 

* 틀 , 뼈대 
  * 어떤 문제를 해결하기 위한 구조의 뼈대가 되는 기본구조.

즉, 일정 타입의 데이터들이 모여 쉽게 가공할 수 있도록 지원하는 

자료구조들의 뼈대(기본구조)   



## 2. 구조

<img src="https://blog.kakaocdn.net/dn/AGpq3/btqI07wkE1A/yX10IjGgt6N3G6rkT1Ievk/img.png" width="450px">

* 점선 : 구현관계 , 실선: 확장 관계(상속).
* 자바에서 제공하는 Collection은 크게 List, Queue , Set 3가지의 인터페이스로 이루어져있다. 그리고 이때까지 학습한 자료구조들은 이를 구현 함으로써 동작한다

세부적인 구현들은 각장에서 학습하기로하자.



❗️Collection 인터페이스가 extends 하고있는 Iterable 인터페이스에 대해서는 iteratora메소드와 for -each loop 와 연관지어 따로 학습하기로 하자.



