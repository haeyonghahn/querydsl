# querydsl
## H2 데이터베이스 설치
개발이나 테스트 용도로 가볍고 편리한 DB, 웹 화면 제공

- https://www.h2database.com/
- 권한 주기 : `chmod 755 h2.sh`
- 데이터베이스 파일 생성 방법
  - `jdbc:h2:~/querydsl` (최소 한번)
  - 이후 부터는 `jdbc:h2:tcp://localhost/~/querydsl` 이렇게 접속

> 주의 : 1.4.200 버전은 몇가지 오류가 있다. 현재 안정화 버전은 1.4.199(2019-03-13) 이다.    
  다운로드 링크: https://www.h2database.com/html/download.html
