# 도커 적용

도커에 대한 개념은 여기서 정리를 했었으니까욤
https://github.com/Afdddd/TIL/blob/main/Docker/Docker%EB%9E%80.md

1. Docker 직접 플젝에 적용해보기 
2. 그 과정에서 생긴 트러블 이슈 정리 <br>
   2_1. docker DB 연결 불가 <br>
   2_2. 네트워크 설정

적용하기 전에 당연히!!! Docker를 다운받으셔야 합니답.<br>
다운받았다는 가정하에 진행 

# 1️⃣ docker file작성

⇒ 도커 파일 작성은 root 경로 바로 아래에 만들어줘야 해용. 

```jsx
# 1. 사용할 자바 버전 기반의 이미지
FROM eclipse-temurin:17-jdk

# 2. 작업 디렉토리를 설정
WORKDIR /app

# 3. 스프링 부트 애플리케이션의 JAR 파일을 컨테이너로 복사
COPY build/libs/본인플젝root이름-0.0.1-SNAPSHOT.jar app.jar

# 4. JAR 파일을 실행
ENTRYPOINT ["java", "-jar", "app.jar"]
```

저는 크게 연결한 라이브러리들이 없기 때문에 위처럼 지정해줬습니당. 

- `WORKDIR` : 도커 컨테이너 안에서 명령어를 실행할 디렉토리를 설정하는 명령어.  ⇒ 위와 같은 경우 이후의 모든 명령어는 /app 디렉토리 안에서 실행됩니당. 
(컨테이너 내부에서 코드와 관련된 파일들이 위치하는 곳)
- `COPY build/libs/ecService-0.0.1-SNAPSHOT.jar app.jar`

⇒ 이거 같은 경우는 저희 앱 build 하면 자동으로 생기고 <br>(gradle 기준임, maven이면 target에 있어용) 그 경로 그대로 써주시면 됩니당. 

그럼 그 다음 도커 이미지를 만들어줘야죵 

# 2️⃣ docker image 만들기

이제 위에 작성한 dockerFile을 토대로 `docker image` 를 만들어 줘야 해요 !! 

### <mark>2_1. 터미널 열기</mark>

(맥북 기준 command + space ⇒ terminal 검색 

```java
docker build -t 만들 이미지 이름 

// 제가 만든 파일 이름 
docker build -t ecService .

```

![스크린샷 2024-08-24 오후 8.35.15.png](/Docker/img/docker1.png)

⇒ 이 오류가 뜨신다면 이미지 이름을 대문자 섞으셔서 그래요. 
ecservice로 바꿨습니당. 

![스크린샷 2024-08-24 오후 8.36.16.png](/Docker/img/docker2.png)

근데 그 다음엔 또 이런 오류가 뜨는데

이거는 사용하려는 Docker 이미지 

`eclipse-temurin:17-jdk-alpine` 가 이 플랫폼에서 지원되지 않는다는 거에여. 그래서 

`eclipse-temurin:17-jdk` 

apline이 아닌 일반 jdk 이미지로 바꿨습니당. 

<span style="color : red;">파일 이미지 완</span>

!!그럼 이제 도커 컨테이너를 실행해봐야게쬬 

### <mark>2_2. 도커 컨테이너 실행하기</mark>

```java
docker run -p 8080:8080 위에서 만든 이미지 이름

//제 이미지예시
docker run -p 8080:8080 ecservice

```

-p 8080:8080은 제 컴퓨터의 8080포트를 도커 컨테이너의 8080번 포트와 연결하겠다는 의미에용. 

이러면 웹 브라우저에서 localhost:8080으로 제 앱에 접속할 수가 있습니당. 

잘 실행된다면 브라우저에 localhost:8080으로 접속해도 잘 뜹니당 

# 그런데 잘 안돼요

## 첫 번째 오류

<aside>
💡 Error creating bean with name 'entityManagerFactory' defined in class path resource [org/springframework/boot/autoconfigure/orm/jpa/HibernateJpaConfiguration.class]: Unable to create requested service [org.hibernate.engine.jdbc.env.spi.JdbcEnvironment] due to: Unable to determine Dialect without JDBC metadata (please set 'jakarta.persistence.jdbc.url' for common cases or 'hibernate.dialect' when a custom Dialect implementation must be provided)

</aside>

⇒ Hibernate가 데이터베이스에 연결할 수 없거나 데이터베이스와 관련된 설정이 누락되어서 뜨는 오류입니당. 

그래서 

1. yml 파일에 hibernate.dialect 추가해주기. 

```java
hibernate.dialect: org.hibernate.dialect.PostgreSQLDialect
```

이렇게 해서 연결은 됐어용 
localhost:8080으로 브라우저에서 접속하면 접속 되더라구용

근데 

![스크린샷 2024-08-24 오후 8.59.06.png](/Docker/img/docker3.png)

postman으로 제 8080에다가 주문데이터 생성을 해보니 

500 error가 나는겁니다. 

**엥  ? 서버 오류가 왜나지..** 

해석해보니까 애플리케이션이 PostgreSQL DB에 연결을 시도했지만, 연결이 거부됐단 뜻. 

제 주문 데이터 생성이 CRUD가 있는데 거기서 postgreSql에 

연결이 안됩나봅니당. 

[https://shawn-dev.oopy.io/463a30bf-bf32-44e4-88be-8b4722e5549a](https://shawn-dev.oopy.io/463a30bf-bf32-44e4-88be-8b4722e5549a) ⇒ 이 글 보고 깨달았는데 

제 애플리케이션에 있는 localhost로 지정한 제 DB url이 도커에서는 본인이 localhost이기 때문에 DB 가 연결이 안되는거였어요… 

# 그래서 어떻게 해결했냐.

### 1. localhost 대신 host.docker.internal

application.yml 파일 

```jsx
url: jdbc:postgresql://host.docker.internal:5432/ecsdb
```

⇒ host.docker.internal 은 Docker에서 제공하는 가상 도메인 이름으로, 주로 Mac과 Windows에서 사용이 됩니당. (Linux는 지원 안함) 
⇒ Docker 컨테이너가 이 도메인을 사용하여 호스트 시스템의 네트워크 자원에 접근할 수 있도록 해주는거에용. 

즉 !! host의 ip가 없어도 접근 가능합니당. 

이 방법은 특정 ip에 종속되지 않으니까 로컬에서 개발할 때 (자주 환경을 바꿀 수 있는 곳에서) 편리하게 사용하기 좋아용. 

다만, 리눅스나 일부 환경에서는 지원되지 않는다는 단점이 있어요.

저는 그래서 그냥 고정 ip방법을 사용했습니당. 

### 2. localhost 대신 ip 적어주기

```jsx
url: jdbc:postgresql://내 ip주소 /DB스키마이름
```

저는 이거 하면서 오류 많이 겪어서 차근차근 단계별로 쓸게용.

2_1. 터미널에 ifconfig 입력해서 사용중인 내 로컬 ip 검색 

```jsx
en0: ~~~
	inet 내 ip 주소 netmask ~~~
```

en0 아래에 있는 ip주소가 본인의 사용중인 로컬 ip 입니당 (Mac 기준)

그런데도 

<aside>
💡  Caused by: org.springframework.beans.factory.BeanCreationException at AbstractAutowireCapableBeanFactory.java:1806

</aside>

⇒ 똑같이 bean 생성 오류가 뜹니당. 여전히 docker에서는 제 postgreSql을 찾지 못하고 있어요.. 

그건 제 postgreSQL `pg_hba.conf` 파일에서 

제 로컬 네트워크 ip 연결을 허용하지 않도록 설정되어 있기 때문입니당. postgreSql의 접근제한 설정은 `pg_hba.conf` 파일에서 관리 되거덩요. 

```java
//일반 적인 경로 
Linux/Mac: /usr/local/var/postgres/pg_hba.conf
Windows: C:\Program Files\PostgreSQL\<version>\data\pg_hba.conf

//그러나 내 경로에는 여기있었다. 
/opt/homebrew/var/postgresql@14/pg_hba.conf
//postgresql@14에는 본인들 버전을 쓰면된다. 
```

[https://blog.naver.com/programming_my00/223559977178](https://blog.naver.com/programming_my00/223559977178) <br>
⇒ 혹시나 경로 찾는 방법이 궁금하시면 참고하세용 

그래서 전 경로를 찾아

```java
vim /opt/homebrew/var/postgresql@14/pg_hba.conf
```

⇒ vim 편집기로 

```java
host    all             all            위에서 찾은 ip주소           md5
```

⇒ 를 추가해주었고 

<aside>

💡 vim 편집기에서 편집하려면 
1. i 를 누른다 (insert 라는 뜻)
2. 수정한다. 
3. 다 수정하면 esc를 누른다. 
4. 저장하려면 :wq를 누른다

</aside>

<br>

```java
brew services restart postgresql  # Mac (Homebrew로 설치된 경우
```

⇒ 설레는 맘으로 재시작을 했으나 

<aside>
💡 org.springframework.beans.factory.BeanCreationException at AbstractAutowireCapableBeanFactory.java:1806
jakarta.persistence.PersistenceException at

</aside>

ㅎㅎ 여전히 beanCreate error

일단 

1️⃣ 제 ip주소에서 오는 모든 요청은 허용하도록 설정했기 때문에 더 이상 pg_hba.conf 문제는 아닙니다. 

2️⃣ postgresql이 제대로 실행중인가  ?  ⇒ 잘 실행중입니다. 

3️⃣ postgresql이 5432 포트에서 수신중인가 ?

```java
sudo lsof -i -P -n | grep LISTEN
```

→ 하면 볼 수 있는데 잘 수신중임. 

# 그럼 이제 문제는 네트워크 뿐이다..

1️⃣ 임의로 로컬에서 postgreSQL 서버에 접근 가능한지 확인

```java
psql -U 제 유저 아이디 -h 제 컴터 ip주소 -d db스키마이름
```

⇒ 이게 가능하다면 외부에서 postgresql에 접근 가능한거거든요 

<aside>
💡 psql: error: connection to server at "내 컴터 ip번호", port 5432 failed: Connection refused
Is the server running on that host and accepting TCP/IP connections?

</aside>

⇒ 역시나 당연히 불가능하죵 

지금 외부에서 접속이 불가능한겁니다. 

외부에서 특정 ip의 요청에 접근 가능하도록 설정하는건 

`postgresql.conf` 파일이거든요 

```java
Mac (Homebrew 설치): /opt/homebrew/var/postgresql@14/postgresql.conf
Linux: /etc/postgresql/<version>/main/postgresql.conf 또는 /var/lib/pgsql/<version>/data/postgresql.conf
Windows: C:\Program Files\PostgreSQL\<version>\data\postgresql.conf
```

다들 본인의 os에 따라서 위치를 찾아보시공 

전 또 vim을 통해 편집기를 열어서 

```java
listen_addresses = 'localhost' // 이던 것을 
listen_addresses = '*' //로 변경 했습니당. 
```

⇒ 참고로 앞에 # 붙어있으면 그건 주석처리니까 #도 지워주셔야해요 (저 첨에 안지워서 적용 안되는거 땜에 엄한 곳에서 이유찾음)

이 설정은 postgreSQL이 로컬 호스트 뿐 아니라, 외부 ip에서도 연결을 수신할 수 있도록 해요. 

```java
brew services restart postgresql@14 
```

설레는 재시작

![스크린샷 2024-08-25 오후 1.02.55.png](/Docker/img/docker4.png)


헤헤 ~!~!~

ㅎㅎ ; ; ; ;; ;;  ; 

헤헤 

-끝- 

다신 보지말자 도커야 
<br>



참고사항 =>
```java
//실행중인 모든 컨테이너 확인 
docker ps 

//실행 중인 컨테이너 중지
docker stop 컨테이너ID

//컨테이너 로그 확인 
docker logs 컨테이너ID
```