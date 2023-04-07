# [실습] 1. JDBC 이해
## 1. 프로젝트 생성
- spring : 3.0, jdk17
- Gradle Project, Package: hello.jdbc, Group: hello, Artifact: jdbc
- Dependecies: spring-boot-starter-jdbc, lombok, h2

## 2. H2 데이터베이스 설정
- h2
```
h2 Version 2.1.214
username: sa
url: (최초)jdbc:h2:~/test, (이후) jdbc:h2:tcp://localhost/~/test
```
- Member table create sql
```
drop table member if exists cascade;
create table member (
                        member_id varchar(10),
                        money integer not null default 0,
                        primary key (member_id)
);
insert into member(member_id, money) values ('hi1',10000);
insert into member(member_id, money) values ('hi2',20000);
```

## 3. JDBC 이해
- jdbc 등장 이유
문제는 각각 데이터베이스 마다 커넥션을 연결하는 방법, SQL을 전달하는 방법, 그리고 결과를 응답 받는 방법 모두 다르다  
크게 2가지문제
> 1) 데이터베이스를 다른 종류의 데이터베이스로 변경시 코드변경  
2) 개발자가 각각의 데이터베이스마다 커넥션 연결, SQL 전달, 그리고 그 결과를 응답받는 방법을 새로 학습
- `JDBC 표준 인터페이스`
JDBC(Java Database Connectivity): 자바에서 데이터베이스에 접속할 수 있도록 하는 자바 API다
> java.sql.Connection, Statement, ResultSet  
자바는 이렇게 표준 인터페이스를 정의해두었다.
