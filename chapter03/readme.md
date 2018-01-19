# 3장 - 템플릿

## Intro

* 1장에서 한 일 다시 보기
  * 초난감 DAO
  * 관심사의 분리
  * 개방폐쇠 원칙(OCP)에 맞게 설계구조 변경
    * 코드에서 어떤부분은 변경을 통해 기능이 다양해지고 확장됨
    * 어떤부분은 고정되어 있으며 변하지 않으려는 성질
    * 구분해서 효율적인 구조를 만들어주는 것

* 템플릿 - 바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법

* 3장의 내용
  * 스프링에 적용된 템플릿 기법을 살펴보기
  * 템플릿을 이용해 완성도 있는 DAO 만들기

## 3.1. 다시보는 초난감 DAO

### 3.1.1. 예외처리 기능을 갖춘 DAO

* 그동안 신경쓰지 않았던 부분 - 예외처리
  * DB 커넥션이라는 제한적인 리소스를 공유해 사용하는 JDBC 코드에는 어떤 이유로든 예외 발생시 사용한 리소스를 반드시 반환하도록 만들어야 함
  * 현재 UserDao의 코드는 중간에 예외 발생시 connection을 닫지 못하는 문제가 있다.
  * connection 자원이 반환되지 못하면 커넥션 풀 자원을 사용하지 못하고 리소스 부족 오류를 내며 서버가 중단되는 사태가 벌어진다.
  * 장시간 사용되는 다중 사용자를 위한 서버에는 치명적이다.
  * 결론 - try / catch / finally 구문으로 감싸고 어떤 상황이든 close()를 부를수 있게 한다.

* JDBC 수정 기능의 예외 처리(코드 참고)
* JDBC 조회 기능의 예외 처리(코드 참고)

## 3.2. 변하는 것과 변하지 않는 것

### 3.2.1. JDBC try / catch / finally 코드의 문제점

* 문제점
  * 복잡한 try/catch/finally 블럭이 2중으로 나옴
  * UserDao 모든 메소드에 반복
* 근시안적인 해결방법 - Copy & Paste
  * 사람의 손으로 편집하다가 실수할 가능성 다분
  * 테스트를 돌려도 잘 된다. 당장에 문제가 없어보이는게 더 큰 문제
  * 옛날 이야기
    * 1주일에 한번씩 DB 커넥션 풀이 꽉 차는 문제발생
    * 전문 컨설턴트의 해결책 - 수백개의 DAO 코드를 A4에 출력해 하나씩 형광펜으로 체크함
    * 몇군데 close() 호출이 잘못된 곳을 찾아 변경

* 그럼 어떻게 효과적으로 해결 해야할까?
  * 변하지 않는, 중복되는 코드와 확장되고 자주 변하는 코드를 분리해야 한다.
  * 1장의 DAO와 DB연결기능 분리와 원리는 같으나 성격이 다르므로 해결 방법도 조금 다르다.

### 3.2.2. 분리와 재사용을 위한 디자인 패턴 적용

* 변하는 부분 
  * SQL 적용부분 
  * 결과를 가져가는 부분
* 변하지 않는 부분
  * try / catch 구문
  * close() 호출

* 메소드 추출
  * 변하는 부분 - 메소드가 추출된 부분
  * 변하지 않는 부분 - 원래 메소드
  * 변하지 않는 부분을 재사용해야 하는데, 변하는 부분이 재사용하기 쉬운 구조가 되었다. 반대로 되어서 적용 실패
* 템플릿 메소드 패턴 적용
  * 상속을 이용한 확장 구조
  * 변하지 않는 부분 - 상위 클래스
  * 변하는 부분 - 하위 클래스
  * Dao의 기능이 생길 때마다 class를 추가해주어야 하는 문제 - 배보다 배꼽이 큼
* 전략 패턴의 적용
  * 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존
  * 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식
  * 변하지 않는 부분 - context(맥락)
  * 변하는 부분 - strategy(전략)
  * 장점
    * 개방 폐쇄 원칙을 잘 지킴
    * 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어남

  ![전략 패턴의 구조](images/3-2.png)

  * deleteAll() 컨텍스트 정리
    * DB 커넥션 가져오기
    * **PreparedStatement를 만들어줄 외부 기능 호출하기 - 전략**
    * 전달받은 PreparedStatement 실행하기
    * 예외가 발생하면 이를 다시 메소드 밖으로 던지기
    * 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기
  * StatementStrategy 라는 인터페이스로 전략을 정의한다.
  ```java
    package springbook.user.dao;
    ...
    public interface StatementStrategy {
        PreparedStatement makePrearedStatement(Connection c) throws SQLException;
    }
  ```
  * 이 인터페이스를 상속하여 실제 바뀌는 코드 부분을 클래스로 만든다.
  ```java
    package springbook.user.dao;
    ...
    public class DeleteAllStatement implements StatementStrategy{
      @Override
      public PreparedStatement makePrearedStatement(Connection c) throws SQLException {
          PreparedStatement ps = c.prepareStatement("delete from users");
          return ps;
      }
    }
  ```
  * 마지막 바뀌지 않는 부분(맥락) 에 적용한다.
  ```java
    public void deleteAll() throws SQLException {
      ...
      try {
          c = dataSource.getConnection();

          StatementStrategy strategy = new DeleteAllStatement();
          ps = strategy.makePrearedStatement(c);

          ps.executeUpdate();
      } catch (SQLException e){
      ...
  ```
  * 하지만 이렇게 Context 안에서 이미 구체적인 전략 클래스인 DeleteAllStatement 가 고정되어 있으면 전략패턴의 취지를 살리지 못함
  * Context 가 어떤 전략을 사용할지 정하는 것은 Context 를 사용하는 Client 가 결정하는게 더 유연한 디자인이 된다.

  ![전략 패턴에서 Client의 역할](images/3-3.png)

  * 위와같은 방식으로 되려면? StatementStrategy를 컨텍스트 메소드의 파라메터로 지정하면 해결된다.

  * 메소드로 분리한 try/catch/finally 컨텍스트 코드
  ```java
    public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
      Connection c = null;
      PreparedStatement ps = null;

      try {
          c = dataSource.getConnection();

          ps = stmt.makePrearedStatement(c);

          ps.executeUpdate();
      ...
  ```
  * 클라이언트 책임을 담당할 deleteAll() 메소드
  ```java
    public void deleteAll() throws SQLException {
        StatementStrategy st = new DeleteAllStatement();
        jdbcContextWithStatementStrategy(st);
    }
  ```

  * 어느정도의 전략패턴의 모습을 갖추었다!

## 3.3. JDBC 전략 패턴의 최적화

### 3.3.1. 전략 클래스의 추가 정보

* UserDao의 add() 메서드 적용
  * add() 도 동일한 과정을 거침
  * 차이점은 add 할 대상 User 오브젝트를 StatementStrategy의 생성자에서 받음