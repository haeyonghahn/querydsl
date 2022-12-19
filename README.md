# querydsl
## 목차
* **[H2 데이터베이스 설치](#H2-데이터베이스-설치)**
* **[기본 문법](#기본-문법)**
  * **[JPQL vs Querydsl](#JPQL-vs-Querydsl)**
  * **[기본 Q-Type 활용](#기본-Q-Type-활용)**
  * **[검색 조건 쿼리](#검색-조건-쿼리)**
  * **[결과 조회](#결과-조회)**
  * **[distinct](#distinct)**
  * **[정렬](#정렬)**
  * **[페이징](#페이징)**
  * **[집합](#집합)**
  * **[조인 - 기본 조인](#조인---기본-조인)**
  * **[조인 - on 절](#조인---on-절)**
  * **[조인 - 페치 조인](#조인---페치-조인)**
  * **[서브 쿼리](#서브-쿼리)**
* **[중급 문법](#중급-문법)**
  * **[프로젝션과 결과 반환 - 기본](#프로젝션과-결과-반환---기본)**
  * **[(중요)프로젝션과 결과 반환 - DTO 조회](#중요프로젝션과-결과-반환---dto-조회)**
  * **[프로젝션과 결과 반환 - @QueryProjection](#프로젝션과-결과-반환---queryprojection)**
  * **[동적 쿼리 - BooleanBuilder 사용](#동적-쿼리---BooleanBuilder-사용)**
  * **[동적 쿼리 - Where 다중 파라미터 사용](#동적-쿼리---Where-다중-파라미터-사용)**
  * **[수정, 삭제 벌크 연산](#수정,-삭제-벌크-연산)**
  * **[SQL function 호출하기](#SQL-function-호출하기)**
* **[실무 활용 - 순수 JPA와 Querydsl](#실무-활용---순수-JPA와-Querydsl)**
  * **[순수 JPA 리포지토리와 Querydsl](#순수-JPA-리포지토리와-Querydsl)**
  * **[동적 쿼리와 성능 최적화 조회 - Builder 사용](#동적-쿼리와-성능-최적화-조회---Builder-사용)**
  * **[동적 쿼리와 성능 최적화 조회 - Where절 파라미터 사용](#동적-쿼리와-성능-최적화-조회---Where절-파라미터-사용)**
  * **[조회 API 컨트롤러 개발](#조회-API-컨트롤러-개발)**

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
### distinct
```java
List<String> result = queryFactory
  .select(member.username).distinct()
  .from(member)
  .fetch();
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
### 조인 - 페치 조인
페치 조인은 SQL에서 제공하는 기능은 아니다. SQL조인을 활용해서 연관된 엔티티를 SQL 한번에 조회하는 기능이다. 주로 성능 최적화에 사용하는 방법이다.      

__페치 조인 미적용__    
지연로딩으로 Member, Team SQL 쿼리 각각 실행
```java
@PersistenceUnit
EntityManagerFactory emf;

@Test
public void fetchJoinNo() {
    em.flush();
    em.clear();

    Member findMember = queryFactory
            .selectFrom(member)
            .where(member.username.eq("member1"))
            .fetchOne();

    boolean loaded = emf.getPersistenceUnitUtil().isLoaded(findMember.getTeam());
    assertThat(loaded).as("패치 조인 미적용").isFalse();
}
```
__페치 조인 적용__
```java
@Test
public void fetchJoinUse() {
    em.flush();
    em.clear();

    Member findMember = queryFactory
            .selectFrom(member)
            .join(member.team, team).fetchJoin()
            .where(member.username.eq("member1"))
            .fetchOne();

    boolean loaded = emf.getPersistenceUnitUtil().isLoaded(findMember.getTeam());
    assertThat(loaded).as("패치 조인 적용").isTrue();
}
```
- `join()`, `leftJoin()` 등 조인 기능 뒤에 fetchJoin() 이라고 추가하면 된다.

### 서브 쿼리
`com.querydsl.jpa.JPAExpressions` 사용
```java
/**
 * 나이가 가장 많은 회원 조회
 */
@Test
public void subQuery() {
    QMember memberSub = new QMember("memberSub");

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.eq(
                    select(memberSub.age.max())
                            .from(memberSub)
            ))
            .fetch();

    assertThat(result).extracting("age")
            .containsExactly(40);
}
```
```java
/**
 * 나이가 평균 나이 이상인 회원
 */
@Test
public void subQueryGoe() {
    QMember memberSub = new QMember("memberSub");

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.goe(
                    select(memberSub.age.avg())
                            .from(memberSub)
            ))
            .fetch();

    assertThat(result).extracting("age")
            .containsExactly(30, 40);
}
```
```java
/**
 * 서브쿼리 여러 건 처리, in 사용
 */
@Test
public void subQueryIn() {
    QMember memberSub = new QMember("memberSub");

    List<Member> result = queryFactory
            .selectFrom(member)
            .where(member.age.in(
                    select(memberSub.age)
                            .from(memberSub)
                            .where(memberSub.age.gt(10))
            ))
            .fetch();

    assertThat(result).extracting("age")
            .containsExactly(20, 30, 40);
}
```
```java
@Test
public void selectSubQuery() {
    QMember memberSub = new QMember("memberSub");

    List<Tuple> fetch = queryFactory
            .select(member.username,
                    select(memberSub.age.avg())
                    .from(memberSub)
            ).from(member)
            .fetch();

    for(Tuple tuple : fetch) {
        System.out.println("username = " + tuple.get(member.username));
    }
}
```
__static import 사용__
```java
import static com.querydsl.jpa.JPAExpressions.select;

List<Member> result = queryFactory
  .selectFrom(member)
  .where(member.age.eq(
         select(memberSub.age.max())
         .from(memberSub)
   ))
   .fetch();
```
__from 절의 서브쿼리 한계__    
JPA JPQL 서브쿼리의 한계점으로 from 절의 서브쿼리(인라인 뷰)는 지원하지 않는다.   
당연히 Querydsl도 지원하지 않는다. 하이버네이트 구현체를 사용하면 select 절의 서브 쿼리는 지원한다. Querydsl도 하이버네이트 구현체를 사용하면
select 절의 서브쿼리를 지원한다.   

__from 절의 서브쿼리 해결방안__
1. 서브쿼리를 join으로 변경한다.(가능한 상황도 있고 불가능한 상황도 있다.)
2. 애플리케이션에서 쿼리를 2번 분리해서 실행한다.
3. nativeSQL을 사용한다.

## 중급 문법
### 프로젝션과 결과 반환 - 기본
프로젝션이란 select절에 뭘 가져올지 대상을 지정하는 것을 말한다.    

__프로젝션 대상이 하나__
```java
@Test
public void simpleProjection() {
    List<String> result = queryFactory
            .select(member.username)
            .from(member)
            .fetch();

    for(String s : result) {
        System.out.println("s = " + s);
    }
}
```
- 프로젝션 대상이 하나면 타입을 명확하게 지정할 수 있음
- 프로젝션 대상이 둘 이상이면 튜플이나 DTO로 조회

__튜플 조회__    
프로젝션 대상이 둘 이상일 때 사용   
`com.querydsl.core.Tuple`
```java
@Test
public void tupleProjection() {
    List<Tuple> result = queryFactory
            .select(member.username, member.age)
            .from(member)
            .fetch();

    for(Tuple tuple : result) {
        String username = tuple.get(member.username);
        Integer age = tuple.get(member.age);
        System.out.println("username = " + username);
        System.out.println("age = " + age);
    }
}
```
> 참고 : QueryDsl에 종속적인 Tuple을 Repository 계층에서 사용하는 것은 괜찮다. 하지만 Repository 계층을 넘어서서 Service 계층이나 Controller까지 넘어가는 것은 
좋은 설계가 아니다. JDBC나 JPA를 비즈니스 로직이 알면 좋지 않은 것처럼. JDBC나 JPA에서 반환해주는 ResultSet을 Repository나 DAO 계층 안에서만 쓰도록 하고 
나머지 계층에서는 의존도가 없도록 하는 것이 좋은 설계이다. 이렇게 사용한다면 하부 기술을 QueryDsl에서 다른 것으로 바꾸더라도 Controller나 Service 계층을 바꿀 필요가 없게 된다.
결국, Tuple은 Repository계층에서 사용하고 Dto로 변환하여 사용하는 것이 올바른 방법이다.

### (중요)프로젝션과 결과 반환 - DTO 조회
__순수 JPA에서 DTO 조회__
```java
@Test
public void findDtoByJPQL() {
//        List<MemberDto> result =
//            em.createQuery("select m from Member m", MemberDto.class)
//            .getResultList(); 타입이 안맞아서 오류 발생.
    List<MemberDto> result = em.createQuery(
    "select new study.querydsl.dto.MemberDto(m.username, m.age)" +
            "from Member m", MemberDto.class)
        .getResultList();

    for(MemberDto memberDto : result) {
        System.out.println("memberDto =" + memberDto);
    }
}
```
- 순수 JPA에서 DTO를 조회할 때는 new 명령어를 사용해야함
- DTO의 package이름을 다 적어줘야해서 지저분함
- 생성자 방식만 지원함

__Querydsl 빈 생성(Bean population)__    
__프로퍼티 접근 - Setter__
```java
@Test
public void findDtoBySetter() {
    List<MemberDto> result = queryFactory
            .select(Projections.bean(MemberDto.class,
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for(MemberDto memberDto : result) {
        System.out.println("memberDto =" + memberDto);
    }
}
```
__필드 직접 접근__
```java
@Test
public void findDtoByField() {
    List<MemberDto> result = queryFactory
            .select(Projections.fields(MemberDto.class,   //필드에 값을 바로 꽂아버린다
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for(MemberDto memberDto : result) {
        System.out.println("memberDto =" + memberDto);
    }
}
```
__별칭이 다를 때__
```java
@Test
public void findUserDto() {
    QMember memberSub = new QMember("memberSub");

    List<UserDto> result = queryFactory
            .select(Projections.fields(UserDto.class,
                    member.username.as("name"),
                    ExpressionUtils.as(JPAExpressions
                        .select(memberSub.age.max())
                        .from(memberSub), "age")
            ))
            .from(member)
            .fetch();

    for(UserDto userDto : result) {
        System.out.println("userDto =" + userDto);
    }
}
```
- 프로퍼티나 필드 접근 생성 방식에서 이름이 다를 때 해결 방안
- `ExpressionUtils.as(source, alias)` : 필드나 서브 쿼리에 별칭 사용
- `username.as("memberName")` : 필드에 별칭 사용

__생성자 사용__
```java
@Test
public void findDtoByConstructor() {
    List<MemberDto> result = queryFactory
            .select(Projections.constructor(MemberDto.class,    //필드 타입이 맞아야한다
                    member.username,
                    member.age))
            .from(member)
            .fetch();

    for(MemberDto memberDto : result) {
        System.out.println("memberDto =" + memberDto);
    }
}
```

### 프로젝션과 결과 반환 - @QueryProjection
```java
@Data
@NoArgsConstructor
public class MemberDto {

    private String username;
    private int age;

    @QueryProjection
    public MemberDto(String username, int age) {
        this.username = username;
        this.age = age;
    }
}
```
```java
@Test
public void findDtoByQueryProjection() {
    List<MemberDto> result = queryFactory
            .select(new QMemberDto(member.username, member.age))
            .from(member)
            .fetch();

    for(MemberDto memberDto : result) {
        System.out.println("memberDto = " + memberDto);
    }
}
```
- `./gradlew compileQuerydsl`
- `QMemberDto` 생성 확인

이 방법은 컴파일러로 타입을 체크할 수 있으므로 가장 안전한 방법이다. 다만 DTO에 QueryDSL 어노테이션을 유지해야 하는 점과 DTO까지 Q파일을 생성해야 하는 단점이 있다.
> 참고 : DTO는 Repository에서 DTO를 조회한 다음에 Service에서도 사용하고 Controller에서도 사용한다. 심지어 API로 반환하기도 한다. 여러 Layer에 걸쳐 사용된다. 그런데 
DTO에 QueryDSL의 어노테이션이 적용되어 있다 보니 순수한 DTO가 아니라는 점이 단점이다. (의존 관계의 문제) 그래서 DTO를 깔끔하게 가져가고 싶을 때 꺼림직한 느낌이 든다.

### 동적 쿼리 - BooleanBuilder 사용
__동적 쿼리를 해결하는 두가지 방식__    
- BooleanBuilder
- Where 다중 파라미터 사용
```java
@Test
public void dynamicQuery_BooleanBuilder() {
    String usernameParam = "member1";
    Integer ageParam = 10;

    List<Member> result = searchMember1(usernameParam, ageParam);
    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember1(String usernameCond, Integer ageCond) {
    BooleanBuilder builder = new BooleanBuilder();
    if(usernameCond != null) {
        builder.and(member.username.eq(usernameCond));
    }
    if(ageCond != null) {
        builder.and(member.age.eq(ageCond));
    }
    return queryFactory
            .selectFrom(member)
            .where(builder)
            .fetch();
}
```
### 동적 쿼리 - Where 다중 파라미터 사용  
```java
@Test
public void dynamicQuery_whereParam() {
    String usernameParam = "member1";
    Integer ageParam = 10;

    List<Member> result = searchMember2(usernameParam, ageParam);
    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember2(String usernameCond, Integer ageCond) {
    return queryFactory
            .selectFrom(member)
//          .where(usernameEq(usernameCond), ageEq(ageCond))
            .where(allEq(usernameCond, ageCond))
            .fetch();
}

private BooleanExpression usernameEq(String usernameCond) {
    return usernameCond != null ? member.username.eq(usernameCond) : null;
}

private BooleanExpression ageEq(Integer ageCond) {
    return ageCond != null ? member.age.eq(ageCond) : null;
}
```
- where 조건에 null 값은 무시된다.
- 메서드를 다른 쿼리에서도 재활용 할 수 있다.
- 쿼리 자체의 가독성이 높아진다.

__조합 가능__
```java
private BooleanExpression allEq(String usernameCond, Integer ageCond) {
    return usernameEq(usernameCond).and(ageEq(ageCond));
}
```
- null 체크는 주의해서 처리해야함

### 수정, 삭제 벌크 연산
__쿼리 한번으로 대량 데이터 수정__
```java
@Test
@Commit
public void bulkUpdate() {
    long count = queryFactory
            .update(member)
            .set(member.username, "비회원")
            .where(member.age.lt(28))
            .execute();

    em.flush();
    em.clear();

    List<Member> result = queryFactory
            .selectFrom(member)
            .fetch();

    for(Member member : result) {
        System.out.println("member = " + member);
    }
}
```
__기존 숫자에 1 더하기__
```java
@Test
public void bulkAdd() {
    long count = queryFactory
            .update(member)
            .set(member.age, member.age.add(1))
            .execute();
}
```
- 곱하기 : `multiply(x)`

__쿼리 한번으로 대량 데이터 삭제__
```java
@Test
public void bulkDelete() {
    long count = queryFactory
            .delete(member)
            .where(member.age.gt(18))
            .execute();
}
```
> 주의 : JPQL 배치와 마찬가지로, 영속성 컨텍스트에 있는 엔티티를 무시하고 실행되기 때문에 배치 쿼리를
실행하고 나면 영속성 컨텍스트를 초기화 하는 것이 안전하다.

### SQL function 호출하기
SQL function은 JPA와 같이 Dialect에 등록된 내용만 호출할 수 있다.    
![image](https://user-images.githubusercontent.com/31242766/208306822-90a9c025-b029-4c66-b30d-f7ca9b276cee.png)   
![image](https://user-images.githubusercontent.com/31242766/208306828-b9338574-ceac-45ff-92ea-cfbe92bca255.png)

"member"라는 단어를 "M"으로 변경하는 replace 함수 사용
```java
@Test
public void sqlFunction() {
    List<String> result = queryFactory
            .select(Expressions.stringTemplate(
                    "function('replace', {0}, {1}, {2})",
                    member.username, "member", "M"))
            .from(member)
            .fetch();

    for(String s : result) {
        System.out.println("s = " + s);
    }
}
```
소문자로 변경해서 비교해라.
```java
@Test
public void sqlFunction2() {
    List<String> result = queryFactory
            .select(member.username)
            .from(member)
//          .where(member.username.eq(
//                  Expressions.stringTemplate("function('lower', {0})", member.username)))
            .where(member.username.eq(member.username.lower()))
            .fetch();

    for(String s : result) {
        System.out.println("s = " + s);
    }
}
```
lower 같은 ansi 표준 함수들은 querydsl이 상당부분 내장하고 있다. 따라서 다음과 같이 처리해도 결과는 같다.
```java
.where(member.username.eq(member.username.lower()))
```

## 실무 활용 - 순수 JPA와 Querydsl
- 순수 JPA 리포지토리와 Querydsl
- 동적쿼리 Builder 적용
- 동적쿼리 Where 적용
- 조회 API 컨트롤러 개발
### 순수 JPA 리포지토리와 Querydsl
__순수 JPA 리포지토리__
```java
package study.querydsl.repository;

import com.querydsl.jpa.impl.JPAQueryFactory;
import org.springframework.stereotype.Repository;
import study.querydsl.entity.Member;

import javax.persistence.EntityManager;
import java.util.List;
import java.util.Optional;

@Repository
public class MemberJpaRepository {

    private final EntityManager em;
    private final JPAQueryFactory queryFactory;

    public MemberJpaRepository(EntityManager em) {
        this.em = em;
        this.queryFactory = new JPAQueryFactory(em);
    }

    public void save(Member member) {
        em.persist(member);
    }

    public Optional<Member> findById(Long id) {
        Member findMember = em.find(Member.class, id);
        return Optional.ofNullable(findMember); //findMember가 null일 수도 있으므로
    }

    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class)
                .getResultList();
    }

    public List<Member> findByUsername(String username) {
        return em.createQuery("select m from Member m where m.username = :username", Member.class)
                .setParameter("username", username)
                .getResultList();
    }
}
```
__순수 JPA 리포지토리 테스트__
```java
package study.querydsl.repository;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import study.querydsl.entity.Member;

import javax.persistence.EntityManager;
import javax.transaction.Transactional;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

@SpringBootTest
@Transactional
class MemberJpaRepositoryTest {

    @Autowired
    EntityManager em;

    @Autowired
    MemberJpaRepository memberJpaRepository;

    @Test
    public void basicTest() {
        Member member = new Member("member1", 10);
        memberJpaRepository.save(member);

        Member findMember = memberJpaRepository.findById(member.getId()).get();
        assertThat(findMember).isEqualTo(member);

        List<Member> result1 = memberJpaRepository.findAll();
        assertThat(result1).containsExactly(member);

        List<Member> result2 = memberJpaRepository.findByUsername("member1");
        assertThat(result2).containsExactly(member);
    }
}
```
__Querydsl 사용__    
__순수 JPA 리포지토리 - Querydsl 추가__   
```java
public List<Member> findAll_Querydsl() {
    return queryFactory
            .selectFrom(member)
            .fetch();
}

public List<Member> findByUsername_Querydsl(String username) {
    return queryFactory
            .selectFrom(member)
            .where(member.username.eq(username))
            .fetch();
}
```
__Querydsl 테스트 추가__
```java
@Test
public void basicQuerydslTest() {
    Member member = new Member("member1", 10);
    memberJpaRepository.save(member);

    Member findMember = memberJpaRepository.findById(member.getId()).get();
    assertThat(findMember).isEqualTo(member);

    List<Member> result1 = memberJpaRepository.findAll_Querydsl();
    assertThat(result1).containsExactly(member);

    List<Member> result2 = memberJpaRepository.findByUsername_Querydsl("member1");
    assertThat(result2).containsExactly(member);
}
```
__JPAQueryFactory 스프링 빈 등록__   
다음과 같이 `JPAQueryFactory` 를 스프링 빈으로 등록해서 주입받아 사용해도 된다.
```java
@Bean
JPAQueryFactory jpaQueryFactory(EntityManager em) {
   return new JPAQueryFactory(em);
}
```
> 참고 : 동시성 문제는 걱정하지 않아도 된다. 왜냐하면 여기서 스프링이 주입해주는 엔티티 매니저는 실제
동작 시점에 진짜 엔티티 매니저를 찾아주는 프록시용 가짜 엔티티 매니저이다. 이 가짜 엔티티 매니저는
실제 사용 시점에 트랜잭션 단위로 실제 엔티티 매니저(영속성 컨텍스트)를 할당해준다. 
더 자세한 내용은 자바 ORM 표준 JPA 책 13.1 트랜잭션 범위의 영속성 컨텍스트를 참고하자.

### 동적 쿼리와 성능 최적화 조회 - Builder 사용
__MemberTeamDto - 조회 최적화용 DTO 추가__   
```java
package study.querydsl.dto;

import com.querydsl.core.annotations.QueryProjection;
import lombok.Data;

@Data
public class MemberTeamDto {

    private Long memberId;
    private String username;
    private int age;
    private Long teamId;
    private String teamName;

    @QueryProjection
    public MemberTeamDto(Long memberId, String username, int age, Long teamId, String teamName) {
        this.memberId = memberId;
        this.username = username;
        this.age = age;
        this.teamId = teamId;
        this.teamName = teamName;
    }
}
```
- `@QueryProjection` 을 추가했다. `QMemberTeamDto` 를 생성하기 위해 `./gradlew compileQuerydsl`
을 한번 실행하자.
> 참고 : `@QueryProjection` 을 사용하면 해당 DTO가 Querydsl을 의존하게 된다. 이런 의존이 싫으면, 
해당 에노테이션을 제거하고, `Projection.bean()`, `fields()`, `constructor()` 을 사용하면 된다.

__회원 검색 조건__   
```java
package study.querydsl.dto;

import lombok.Data;

@Data
public class MemberSearchCondition {
    //회원명, 팀명, 나이(ageGoe, ageLoe)
    private String username;
    private String teamName;
    private Integer ageGoe;
    private Integer ageLoe;
    
}
```
- 이름이 너무 길면 `MemberCond` 등으로 줄여 사용해도 된다.

__동적쿼리 - Builder 사용__   
```java
import static org.springframework.util.StringUtils.hasText;

public List<MemberTeamDto> searchByBuilder(MemberSearchCondition condition) {
    BooleanBuilder builder = new BooleanBuilder();
    //프론트에서 넘어오는 값이 null 또는 "" 식으로 넘어오는 것을 해결해준다.(StringUtils)
    if(hasText(condition.getUsername())) {
       builder.and(member.username.eq(condition.getUsername()));
    }
    if(hasText(condition.getTeamName())) {
        builder.and(team.name.eq(condition.getTeamName()));
    }
    if(condition.getAgeLoe() != null) {
        builder.and(member.age.goe(condition.getAgeGoe()));
    }
    if(condition.getAgeLoe() != null) {
        builder.and(member.age.loe(condition.getAgeLoe()));
    }

    return queryFactory
            .select(new QMemberTeamDto(
                    member.id.as("memberId"),
                    member.username,
                    member.age,
                    team.id.as("teamId"),
                    team.name.as("teamName")
            ))
            .from(member)
            .leftJoin(member.team, team)
            .where(builder)
            .fetch();
    }
```
> 오류 정정 : 강의 영상에서는 member.id.as("memberId") 라고 적었는데, QMemberTeamDto 는 생성자를 사용하기
때문에 필드 이름을 맞추지 않아도 된다. 따라서 member.id 만 적으면 된다.

__조회 예제 테스트__    
```java
@Test
public void searchTest() {
    Team teamA = new Team("teamA");
    Team teamB = new Team("teamB");
    em.persist(teamA);
    em.persist(teamB);

    Member member1 = new Member("member1", 10, teamA);
    Member member2 = new Member("member2", 20, teamA);

    Member member3 = new Member("member3", 30, teamB);
    Member member4 = new Member("member4", 40, teamB);
    em.persist(member1);
    em.persist(member2);
    em.persist(member3);
    em.persist(member4);

    MemberSearchCondition condition = new MemberSearchCondition();
    condition.setAgeGoe(35);
    condition.setAgeLoe(40);
    condition.setTeamName("teamB");

    List<MemberTeamDto> result = memberJpaRepository.searchByBuilder(condition);

    assertThat(result).extracting("username").containsExactly("member4");
}
```
### 동적 쿼리와 성능 최적화 조회 - Where절 파라미터 사용
__Where절에 파라미터를 사용한 예제__   
```java
public List<MemberTeamDto> search(MemberSearchCondition condition) {
    return queryFactory
            .select(new QMemberTeamDto(
                    member.id.as("memberId"),
                    member.username,
                    member.age,
                    team.id.as("teamId"),
                    team.name.as("teamName")
            ))
            .from(member)
            .leftJoin(member.team, team)
            .where(
                    usernameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe())
            )
            .fetch();
}

private BooleanExpression usernameEq(String username) {
    return hasText(username) ? member.username.eq(username) : null;
}

private BooleanExpression teamNameEq(String teamName) {
    return hasText(teamName) ? team.name.eq(teamName) : null;
}

private BooleanExpression ageGoe(Integer ageGoe) {
    return ageGoe != null ? member.age.goe(ageGoe) : null;
}

private BooleanExpression ageLoe(Integer ageLoe) {
    return ageLoe != null ? member.age.loe(ageLoe) : null;
}
```
__조회 예제 테스트__
```java
@Test
public void searchTest() {
    Team teamA = new Team("teamA");
    Team teamB = new Team("teamB");
    em.persist(teamA);
    em.persist(teamB);

    Member member1 = new Member("member1", 10, teamA);
    Member member2 = new Member("member2", 20, teamA);

    Member member3 = new Member("member3", 30, teamB);
    Member member4 = new Member("member4", 40, teamB);
    em.persist(member1);
    em.persist(member2);
    em.persist(member3);
    em.persist(member4);

    MemberSearchCondition condition = new MemberSearchCondition();
    condition.setAgeGoe(35);
    condition.setAgeLoe(40);
    condition.setTeamName("teamB");

    List<MemberTeamDto> result = memberJpaRepository.search(condition);

    assertThat(result).extracting("username").containsExactly("member4");
}
```
### 조회 API 컨트롤러 개발
편리한 데이터 확인을 위해 샘플 데이터를 추가하자.   
샘플 데이터 추가가 테스트 케이스 실행에 영향을 주지 않도록 다음과 같이 프로파일을 설정하자.   

__프로파일 설정__   
`src/main/resources/application.yml`
```yml
spring:
 profiles:
 active: local
```
테스트는 기존 application.yml을 복사해서 다음 경로로 복사하고, 프로파일을 test로 수정하자   
`src/test/resources/application.yml`
```yml
spring:
 profiles:
 active: test
```
이렇게 분리하면 main 소스코드와 테스트 소스 코드 실행시 프로파일을 분리할 수 있다.

__샘플 데이터 추가__   
```java
package study.querydsl.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;
import study.querydsl.entity.Member;
import study.querydsl.entity.Team;

import javax.annotation.PostConstruct;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.transaction.Transactional;

@Profile("local")
@Component
@RequiredArgsConstructor
public class InitMember {

    private final InitMemberService initMemberService;

    @PostConstruct
    public void init() {
        initMemberService.init();
    }

    @Component
    static class InitMemberService {
        @PersistenceContext
        private EntityManager em;

        @Transactional
        public void init() {
            Team teamA = new Team("teamA");
            Team teamB = new Team("teamB");
            em.persist(teamA);
            em.persist(teamB);

            for(int i = 0; i < 100; i++) {
                Team selectedTeam = i % 2 == 0 ? teamA : teamB;
                em.persist(new Member("member"+i, i, selectedTeam));
            }
        }
    }
}
```
__조회 컨트롤러__
```java
package study.querydsl.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import study.querydsl.dto.MemberSearchCondition;
import study.querydsl.dto.MemberTeamDto;
import study.querydsl.repository.MemberJpaRepository;

import java.util.List;

@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberJpaRepository memberJpaRepository;

    @GetMapping("/v1/members")
    public List<MemberTeamDto> searchMemberV1(MemberSearchCondition condition) {
        return memberJpaRepository.search(condition);
    }
}
```
- 예제 실행(postman)
- `http://localhost:8080/v1/members?teamName=teamB&ageGoe=31&ageLoe=35`
