# querydsl
## 목차
* **[H2 데이터베이스 설치](#H2-데이터베이스-설치)**
* **[기본 문법](#기본-문법)**
  * **[JPQL vs Querydsl](#JPQL-vs-Querydsl)**
  * **[기본 Q-Type 활용](#기본-Q-Type-활용)**
  * **[검색 조건 쿼리](#검색-조건-쿼리)**
  * **[결과 조회](#결과-조회)**
  * **[정렬](#정렬)**
  * **[페이징](#페이징)**
  * **[집합](#집합)**
  * **[조인 - 기본 조인](#조인---기본-조인)**
  * **[조인 - on 절](#조인---on-절)**

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
__기본 검색 쿼리__
```java
@Test
public void search() {
 Member findMember = queryFactory
      .selectFrom(member)
      .where(member.username.eq("member1")
      .and(member.age.eq(10)))
      .fetchOne();
      
 assertThat(findMember.getUsername()).isEqualTo("member1");
}
```
- 검색 조건은 `.and()`, `or()`를 메서드 체인으로 연결할 수 있다.

> 참고 : `select`, `from`을 `selectFrom`으로 합칠 수 있음

__JPQL이 제공하는 모든 검색 조건 제공__
```java
member.username.eq("member1") // username = 'member1'
member.username.ne("member1") //username != 'member1'
member.username.eq("member1").not() // username != 'member1'
member.username.isNotNull() //이름이 is not null
member.age.in(10, 20) // age in (10,20)
member.age.notIn(10, 20) // age not in (10, 20)
member.age.between(10,30) //between 10, 30
member.age.goe(30) // age >= 30
member.age.gt(30) // age > 30
member.age.loe(30) // age <= 30
member.age.lt(30) // age < 30
member.username.like("member%") //like 검색
member.username.contains("member") // like ‘%member%’ 검색
member.username.startsWith("member") //like ‘member%’ 검색
...
```
__AND 조건을 파라미터로 처리__
```java
@Test
public void searchAndParam() {
 List<Member> result1 = queryFactory
   .selectFrom(member)
   .where(member.username.eq("member1"),
          member.age.eq(10))
   .fetch();
 
 assertThat(result1.size()).isEqualTo(1);
}
```
- `where()`에 파라미터로 검색조건을 추가하면 `AND`조건이 추가됨
- 이 경우 `null`값은 무시 -> 메서드 추출을 활용해서 동적 쿼리를 깔끔하게 만들 수 있음

### 결과 조회
- `fetch()` : 리스트 조회, 데이터 없으면 빈 리스트 반환
- `fetchOne()` : 단 건 조회
 - 결과가 없으면 : `null`
 - 결과가 둘 이상이면 : `com.querydsl.core.NoUniqueResultException`
- `fetchFirst()` : `limit(1).fetchOne()`
- `fetchResults()` : 페이징 정보 포함, total count 쿼리 추가 실행
- `fetchCount()` : count 쿼리로 변경해서 count 수 조회
```java
@Test
public void resultFetch() {
    List<Member> fetch = queryFactory
              .selectFrom(member)
              .fetch();

    Member fetchOne = queryFactory
              .selectFrom(QMember.member)
              .fetchOne();

    Member fetchFirst = queryFactory
              .selectFrom(member)
              .fetchFirst();

    QueryResults<Member> results = queryFactory
            .selectFrom(member)
            .fetchResults();

    results.getTotal();
    List<Member> content = results.getResults();

    long count = queryFactory
            .selectFrom(member)
            .fetchCount();

    //페이징 쿼리가 복잡해지면 content를 가져오는 쿼리와 실제 totalCount를 가져오는 쿼리가
    //성능으로 인해 다를 때가 있다. 성능으로 인해서 totalCount를 가져오는 것이 심플해지면서 발생하는 현상이다.
    //그래서 이때는 totalCount를 가져오는 것과 content를 가져오는 쿼리 두 방을 따로 날려야 한다.
}
```
### 정렬
```java
/**
 * 회원 정렬 순서
 * 1. 회원 나이 내림차순(desc)
 * 2. 회원 이름 올림차순(asc)
 * 단, 2에서 회원 이름이 없으면 마지막에 출력(nulls last)
 */
@Test
public void sort() {
    em.persist(new Member(null, 100));
    em.persist(new Member("member5", 100));
    em.persist(new Member("member6", 100));

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.eq(100))
            .orderBy(member.age.desc(), member.username.asc().nullsLast())
            .fetch();

    Member member5 = result.get(0);
    Member member6 = result.get(1);
    Member memberNull = result.get(2);
    assertThat(member5.getUsername()).isEqualTo("member5");
    assertThat(member6.getUsername()).isEqualTo("member6");
    assertThat(memberNull.getUsername()).isNull();
}
```
- `desc()`, `asc()` : 일반 정렬
- `nullsLast()`, `nullsFirst()` : null 데이터 순서 부여

### 페이징
__조회 건수 제한__
```java
@Test
public void paging1() {
    List<Member> result = queryFactory
            .selectFrom(member)
            .orderBy(member.username.desc())
            .offset(1) //0부터 시작(zero index)
            .limit(2) //최대 2건 조회
            .fetch();

    assertThat(result.size()).isEqualTo(2);
}
```
__전체 조회 수가 필요하면?__
```java
@Test
public void paging2() {
    QueryResults<Member> queryResults = queryFactory
            .selectFrom(member)
            .orderBy(member.username.desc())
            .offset(1) //0부터 시작(zero index)
            .limit(2) //최대 2건 조회
            .fetchResults();

    assertThat(queryResults.getTotal()).isEqualTo(4);
    assertThat(queryResults.getLimit()).isEqualTo(2);
    assertThat(queryResults.getOffset()).isEqualTo(1);
    assertThat(queryResults.getResults().size()).isEqualTo(2);
}
```
> 주의 : count 쿼리가 실행되니 성능상 주의!

> 참고 : 실무에서 페이징 쿼리를 작성할 때, 데이터를 조회하는 쿼리는 여러 테이블을 조인해야 하지만,
count 쿼리는 조인이 필요 없는 경우도 있다. 그런데 이렇게 자동화된 count 쿼리는 원본 쿼리와 같이 모두 조인을 해버리기 때문에 
성능이 안나올 수 있다. count 쿼리에 조인이 필요없는 성능 최적화가 필요하다면, count 전용 쿼리를 별도로 작성해야 한다.

### 집합
__집합 함수__
```java
@Test
public void aggregation() {
    List<Tuple> result = queryFactory
            .select(member.count(),
                    member.age.sum(),
                    member.age.avg(),
                    member.age.max(),
                    member.age.min())
            .from(member)
            .fetch();

    Tuple tuple = result.get(0);
    assertThat(tuple.get(member.count())).isEqualTo(4);
    assertThat(tuple.get(member.age.sum())).isEqualTo(100);
    assertThat(tuple.get(member.age.avg())).isEqualTo(25);
    assertThat(tuple.get(member.age.max())).isEqualTo(40);
    assertThat(tuple.get(member.age.min())).isEqualTo(10);
}
```
__GroupBy 사용__
```java
/**
 * 팀의 이름과 각 팀의 평균 연령을 구해라.
 */
@Test
public void group() throws Exception {
    List<Tuple> result = queryFactory
            .select(team.name, member.age.avg())
            .from(member)
            .join(member.team, team)
            .groupBy(team.name)
            .fetch();

    Tuple teamA = result.get(0);
    Tuple teamB = result.get(1);

    assertThat(teamA.get(team.name)).isEqualTo("teamA");
    assertThat(teamA.get(member.age.avg())).isEqualTo(15);

    assertThat(teamB.get(team.name)).isEqualTo("teamB");
    assertThat(teamB.get(member.age.avg())).isEqualTo(35);
}
```
`groupBy`, 그룹화된 결과를 제한하려면 `having`    

__groupBy(), having() 예시__
```java
...
.groupBy(item.price)
.having(item.price.gt(1000))
```
### 조인 - 기본 조인
__기본 조인__
조인의 기본 문법은 첫 번째 파라미터에 조인 대상을 지정하고, 두 번째 파라미터에 별칭(alias)으로 사용할 Q 타입을 지정하면 된다.
```java
join(조인 대상, 별칭으로 사용할 Q타입)
```
```java
/**
 * 팀 A에 소속된 모든 회원
 */
@Test
public void join() {
    List<Member> result = queryFactory
            .selectFrom(member)
            .join(member.team, team)
            .where(team.name.eq("teamA"))
            .fetch();

    assertThat(result)
            .extracting("username")
            .containsExactly("member1", "member2");
}
```
- `join()`, `innerJoin()` : 내부 조인(inner join)
- `leftJoin()` : left 외부 조인(left outer join)
- `rightJoin()` : right 외부 조인(right outer join)
- JPQL의 `on`과 성능 최적화를 위한 `fetch` 조인 제공

__세타 조인__     
연관관계가 없는 필드로 조인
```java
/**
 * 세타 조인(연관관계가 없는 필드로 조인)
 * 회원의 이름이 팀 이름과 같은 회원 조회
 */
@Test
public void theta_join() {
    em.persist(new Member("teamA"));
    em.persist(new Member("teamB"));

    List<Member> result = queryFactory
            .select(member)
            .from(member, team)
            .where(member.username.eq(team.name))
            .fetch();

    assertThat(result)
            .extracting("username")
            .containsExactly("teamA", "teamB");
}
```
- from 절에 여러 엔티티를 선택해서 세타 조인
- 외부 조인 불가능

### 조인 - on 절
- ON절을 활용한 조인(JPA 2.1부터 지원)
1. 조인 대상 필터링
2. 연관관계 없는 엔티티 외부 조인

__1. 조인 대상 필터링__
```java
/**
 * 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회
 * JPQL : SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'teamA'
 * SQL : SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID = t.id and t.name = 'teamA'
 */
@Test
public void join_on_filtering() {
    List<Tuple> result = queryFactory
            .select(member, team)
            .from(member)
            .leftJoin(member.team, team).on(team.name.eq("teamA"))
            .fetch();

    for(Tuple tuple : result) {
        System.out.println("tuple = " + tuple);
    }
}
```
__결과__
```console
t=[Member(id=3, username=member1, age=10), Team(id=1, name=teamA)]
t=[Member(id=4, username=member2, age=20), Team(id=1, name=teamA)]
t=[Member(id=5, username=member3, age=30), null]
t=[Member(id=6, username=member4, age=40), null]
```
> 참고 : on 절을 활용해 조인 대상을 필터링 할 때, 외부조인이 아니라 내부조인(inner join)을 사용하면,
where 절에서 필터링 하는 것과 기능이 동일하다. 따라서 on 절을 활용한 조인 대상 필터링을 사용할 때,
내부조인이면 익숙한 where 절로 해결하고, 정말 외부조인이 필요한 경우에만 이 기능을 사용하자.

__2. 연관관계 없는 엔티티 외부 조인__
```java
/**
 * 연관관계 없는 엔티티 조인
 * 예) 회원의 이름과 팀의 이름이 같은 대상 외부 조인
 * JPQL : SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name
 * SQL : SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.name
 */
@Test
public void join_on_no_relation() {
    em.persist(new Member("teamA"));
    em.persist(new Member("teamB"));

    List<Tuple> result = queryFactory
            .select(member, team)
            .from(member)
            .leftJoin(team).on(member.username.eq(team.name))
            .fetch();

    for(Tuple tuple : result) {
        System.out.println("t=" + tuple);
    }
}
```
- 하이버네이트 5.1부터 `on`을 사용해서 서로 관계가 없는 필드로 외부 조인하는 기능이 추가되었다. 물론 내부 조인도 가능하다.
- 주의! 문법을 잘 봐야 한다. leftJoin() 부분에 일반 조인과 다르게 엔티티 하나만 들어간다.
  - 일반조인 : `leftJoin(member.team, team)`
  - on조인 : `from(member).leftJoin(team).on(xxx)`
  
__결과__
```console
t=[Member(id=3, username=member1, age=10), null]
t=[Member(id=4, username=member2, age=20), null]
t=[Member(id=5, username=member3, age=30), null]
t=[Member(id=6, username=member4, age=40), null]
t=[Member(id=7, username=teamA, age=0), Team(id=1, name=teamA)]
t=[Member(id=8, username=teamB, age=0), Team(id=2, name=teamB)]
```
