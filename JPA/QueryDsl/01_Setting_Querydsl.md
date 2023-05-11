[실전! Querydsl 대시보드 - 인프런 | 강의 (inflearn.com)](https://www.inflearn.com/course/querydsl-%EC%8B%A4%EC%A0%84/dashboard)



# 01 - Querydsl 설정

> Querydsl 은 스프링 부트 프로젝트 생성시 자동으로 주입받거나 할 수없다.



#### build.gradle

```groovy
buildscript {
	ext {
		queryDslVersion = "5.0.0"
	}
}

plugins {
	id 'java'
	id 'org.springframework.boot' version '2.7.11'
	id 'io.spring.dependency-management' version '1.0.15.RELEASE'
	id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
}

group = 'study'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-web'

	implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.5.8'

	//querydsl 추가
	implementation "com.querydsl:querydsl-jpa:${queryDslVersion}"
	annotationProcessor "com.querydsl:querydsl-apt:${queryDslVersion}"

	compileOnly 'org.projectlombok:lombok'
	runtimeOnly 'com.h2database:h2'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'

}

tasks.named('test') {
	useJUnitPlatform()
}

//querydsl 추가 시작
def querydslDir = "$buildDir/generated/querydsl"

querydsl {
	jpa = true
	querydslSourcesDir = querydslDir
}
sourceSets {
	main.java.srcDir querydslDir
}
configurations {
	querydsl.extendsFrom compileClasspath
}
compileQuerydsl {
	options.annotationProcessorPath = configurations.querydsl
}
//이부분이 ide 버전에따라 설정이 바뀐다 이때 안된다고 좌절 하지말고
//구글링으로 전 오지게 굽다보면 됨.
//querydsl 추가 끝
```

+ 스프링 부트 2.6x 버전 이상 사용시  Querydsl 5.0 을 사용한다.

+ 따라서 최상단에서 확인 할 수 있듯이 버전을 직접 명시 해주었다. -> 명시안하면 제대로 작동안함.

```java
def querydslDir = "$buildDir/generated/querydsl"
```

이부분에서 Q타입이 저장되는 위치를 지정했다.

Q타입은 컴파일 시점에 자동생성된다. 버전관리에는 포함 하지 않는 것이 좋다.



> ***Q타입 수동 설정 ***
> 
> : Gradle -> Tasks ->  other -> compileQuerydsl



+ 참고로 보통 build 폴더는 git에 포함하지 않는다. (gitignore)

<br>



#### application.yml

```yaml
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/querydsl
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
    # show_sql: true
        format_sql: true
        use_sql_comments: true #JPQL을 보여줌.
logging.level:
  org.hibernate.SQL: debug
# org.hibernate.type: trace
```


