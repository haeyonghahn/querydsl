# querydsl
## 목차
* **[H2 데이터베이스 설치](#H2-데이터베이스-설치)**
* **[기본 문법](#기본-문법)**
  * **[JPQL vs Querydsl](#JPQL-vs-Querydsl)**
  * **[기본 Q-Type 활용](#기본-Q-Type-활용)**
  * **[검색 조건 쿼리](#검색-조건-쿼리)**

## H2 데이터베이스 설치
개발이나 테스트 용도로 가볍고 편리한 DB, 웹 화면 제공

- https://www.h2database.com/
- 권한 주기 : `chmod 755 h2.sh`
- 데이터베이스 파일 생성 방법
  - `jdbc:h2:~/querydsl` (최소 한번)
  - 이후 부터는 `jdbc:h2:tcp://localhost/~/querydsl` 이렇게 접속

> 주의 : 1.4.200 버전은 몇가지 오류가 있다. 현재 안정화 버전은 1.4.199(2019-03-13) 이다.    
  다운로드 링크: https://www.h2database.com/html/download.html

## 기본 문법
### JPQL vs Querydsl
- `Parameter 바인딩` : JPQL은 setParameter로 파라미터를 바인딩해준다. QueryDsl은 JDBC에 있는 preparedstatement로 파라미터 바인딩을 자동으로 한다.
- `오류 시점` : JPQL은 쿼리가 문자로 작성되기 떄문에 실행 시점에 오류가 발생하게 된다. 하지만 QueryDsl은 컴파일 시점에 오류를 잡아주게 된다.
```java
@Test
public void startJPQL() {
    //member1을 찾아라.
    String qlString =
            "select m from Member m " +
            "where m.username = :username";

    Member findMember = em.createQuery(qlString, Member.class)
            .setParameter("username", "member1")
            .getSingleResult();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}
```
```java
@Test
public void startQuerydsl() {
    //member1을 찾아라.
    //other -> compileQueryDsl 실행
    //build -> generated
    QMember m = new QMember("m");

    Member findMember = queryFactory
            .select(m)
            .from(m)
            .where(m.username.eq("member1"))    //파라미터 바인딩 처리
            .fetchOne();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}
```
### 기본 Q-Type 활용
#### Q클래스 인스턴스를 사용하는 2가지 방법
```java
QMember qMember = new QMember("m"); //별칭 직접 지정
QMember qMember = QMember.member; //기본 인스턴스 사용
```
#### 기본 인스턴스를 static import와 함께 사용
```java
import static study.querydsl.entity.QMember.*;

@Test
public void startQuerydsl() {
    //member1을 찾아라.
    //other -> compileQueryDsl 실행
    //build -> generated

    Member findMember = queryFactory
            .select(member)
            .from(member)
            .where(member.username.eq("member1"))    //파라미터 바인딩 처리
            .fetchOne();

    assertThat(findMember.getUsername()).isEqualTo("member1");
}
```
QueryDsl은 JPQL의 빌더 역할을 하는 도구일 뿐이다. 결과적으로 QueryDsl 코드로 작성한 것은 JPQL이 된다고 볼 수 있다. 다음 설정을 추가하면 실행되는 JPQL을 볼 수 있다.
```yml
spring.jpa.properties.hibernate.use_sql_comments: true
```
![image](https://user-images.githubusercontent.com/31242766/206718739-39d5dca4-bce3-4d25-a1d7-79ec32169adf.png)

![image](https://user-images.githubusercontent.com/31242766/206718804-b00c38ac-05d2-41c3-b60a-3360bb7d482f.png)

> Q클래스 인스턴스는 언제 사용할까?   
만약, 같은 테이블을 조인할 경우에 alias가 같으면 안되므로 Q클래스 인스턴스를 사용하자. 그렇지 않은 경우에는 static impoart를 사용하여 코드를 깔끔히 정리하자.

### 검색 조건 쿼리
