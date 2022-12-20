---
title:  "[Java] HikariCP DB ConnectionTimeout에 대한 Troubleshooting"

categories: Java

toc: true
toc_sticky: true

date: 2022-12-20
last_modified_at: 2022-12-20
---

# HikariCP DB ConnectionTimeout에 대한 Troubleshooting


업무를 보던중 사내 API서버가 죽는 일이 벌어졌습니다.  
다수의 서버가 존재하지만 하나의 서버만 CPU가 급격히 증가하더니 뻗어버렸습니다.


> 갑자기 무수한 요청이 오더니 죽어버린 서버..

![Connection Poll]({{site.url}}/assets/image/2022-12/21-trouble001.png)


무슨일인고.. 하고 서버로그를 보니, HikariCP가 Connecion을 얻을 수 없어서 timeoutException을 발생시켰다는 로그를 발견했습니다.

```shell
java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30010ms.
	at com.zaxxer.hikari.pool.HikariPool.createTimeoutException(HikariPool.java:696) ~[HikariCP-4.0.3.jar:na]
	at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:197) ~[HikariCP-4.0.3.jar:na]
	at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:162) ~[HikariCP-4.0.3.jar:na]
	at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:128) ~[HikariCP-4.0.3.jar:na]
	at org.hibernate.engine.jdbc.connections.internal.DatasourceConnectionProviderImpl.getConnection(DatasourceConnectionProviderImpl.java:122) ~[hibernate-core-5.6.7.Final.jar:5.6.7.Final]
	at org.hibernate.internal.NonContextualJdbcConnectionAccess.obtainConnection(NonContextualJdbcConnectionAccess.java:38) ~[hibernate-core-5.6.7.Final.jar:5.6.7.Final]
	at org.hibernate.resource.jdbc.internal.LogicalConnectionManagedImpl.acquireConnectionIfNeeded(LogicalConnectionManagedImpl.java:108) ~[hibernate-core-5.6.7.Final.jar:5.6.7.Final]
	at org.hibernate.resource.jdbc.internal.LogicalConnectionManagedImpl.getPhysicalConnection(LogicalConnectionManagedImpl.java:138) ~[hibernate-core-5.6.7.Final.jar:5.6.7.Final]
	at org.hibernate.internal.SessionImpl.connection(SessionImpl.java:516) ~[hibernate-core-5.6.7.Final.jar:5.6.7.Final]
	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.beginTransaction(HibernateJpaDialect.java:152) ~[spring-orm-5.3.17.jar:5.3.17]
	at org.springframework.orm.jpa.JpaTransactionManager.doBegin(JpaTransactionManager.java:421) ~[spring-orm-5.3.17.jar:5.3.17]
...
```


## 첫번째 HikariCP 튜닝

처음 이슈트래킹시 한서버에 급격한 요청으로 인해, API 서버의 Connection Pool에서 Connection이 부족해 일어나는 현상으로 파악했습니다.  

물론 다시 생각해보면 위에 생각은 틀린 생각이었습니다.

수많은 요청이 들어와 Connection이 부족하게 된다면 다른 서버에도 영향이 갔을것인데 **하나의 서버에만 문제가 생겼다는 점**과 그렇다고 또 **요청의 수가 서버가 죽을정도로 많지는 않았다는 점**

그외 몇가지로 인해 찜찜함을 가지고, HikariCP의 설정을 확인 해봤습니다.

먼저 한가지 원인중 하나로 봤던것 중 하나는 사내에서 사용하는 MySql의 `wait_timeout`의 시간과 HiakriCP에서 사용하는 `maxLifetime`에서 차이에서 있었습니다.

사내 MySql의 표준 `wait_timeout`의 설정은 25초로 HikariCP 설정의 `maxLifetime` 최소값보다 적었기 때문에 항상 HikariCP가 Connection을 반환하기도 전에 MySql에서 반납하는 현상이 벌어졌었습니다.

```shell
2022-12-20 22:56:43.925 DEBUG 20016 --- [           main] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Failed to validate connection com.mysql.jdbc.JDBC4ReplicationMySQLConnection@35e11917 (No operations allowed after connection closed.). 
Possibly consider using a shorter maxLifetime value.
```

(결론은 아니었습니다 ㅎㅎ)

설정을 확인해보니, 기존의 Connection 설정이 요청의 수에 조금 부족한 감이 있었고 HikariCP의 설정을 변경한 뒤 경과를 지켜보도록 하였습니다.

```yaml
spring:
  datasource:
    driver-class-name: 
    url: 
    username: 
    password:
    hikari:
      maximum-pool-size: 20
      max-lifetime: 30000
      connection-timeout: 10000
      validation-timeout: 10000
```