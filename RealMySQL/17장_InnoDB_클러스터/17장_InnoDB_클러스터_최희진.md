## 3. MySQL Shell

- MySQL 셸은 고급 클라이언트 툴로, SQL, Javascript, Python 모드를 지원한다.

  ```bash
  \py
  Switching to Python mode...
  
  \sql
  Switching to SQL mode...
 
  \js
  Switching to JavaScript mode...
  ```

- MySQL 서버에 대해 쉽고 편리하게 작업할 수 있도록 API를 제공한다.
    - `XDevAPI`: MySQL 서버에서 관계형 데이터와 문서 기반 데이터를 모두 처리할 수 있게 해주는 API
    - `AdminAPI`: MySQL 서버의 설정을 변경하고 InnoDB 클러스터 및 InnoDB 레플리카셋을 구축할 수 있게 해주는 API

- 사용자는 MySQL 셸에 내장돼 있는 글로벌 객체들과 각 객체에 구현돼 있는 메서드를 통해 API를 사용할 수 있다.
    - 글로벌 객체는 Javascript와 Python 모드에서만 사용 가능하다.

      | 글로벌 객체 | 설명 |
          | --- | --- |
      | session | MySQL 서버에 연결했을 때 생성된 세션에 매핑되는 객체로, 세션 단위에서 사용할 수 있는 기능 제공 |
      | dba | InnoDB 클러스터 및 InnoDB 레플리카셋 구축과 관련된 기능 제공 |
      | cluster | InnoDB 클러스터에 매핑되는 객체로, 클러스터 설정 변경 등 클러스터 관련 기능 제공 |
      | rs | InnoDB 레플리카셋에 매핑되는 객체로, 레플리카셋 설정 변경 등 레플리카셋 관련 기능 제공 |
      | db | X 프로토콜을 사용해서 MySQL에 연결한 경우 연결 시 지정했던 데이터베이스에 매핑되는 객체로, 데이터베이스 관련 기능을 제공 |
      | shell | MySQL 셸 설정 변경 등  셸과 관련된 기능 제공 |
      | util | MySQL 서버의 업그레이드, 데이터 로딩 또는 추출 등 유용한 작업 기능 제공 |

## 4. MySQL 라우터

- MySQL 라우터는 InnoDB 클러스터에서 애플리케이션 서버로부터 유입된 요청을 클러스터 내 적절한 MySQL 서버로 전달하고 반환하는 프락시 역할을 한다.
  ![](https://velog.velcdn.com/images/prologue/post/e3a4ac52-bdf8-466a-bbf3-e65fba5f67df/image.jpg)

#### MySQL 라우터의 기능
- **InnoDB 클러스터의 MySQL 구성 변경 자동 감지**
  클러스터의 MySQL 서버 구성이 변경되면 MySQL 라우터느 갱신된 정보로 자동으로 감지하므로 애플리세이션 서버의 커넥션 설정 정보를 변경할 필요가 없다.
- **쿼리 부하 분산**
  MySQL 라우터는 단순히 쿼리들을 클러스터 내 MySQL 서버로 전달만 하는 것이 아니라 여러 MySQL에 나눠서 처리되도록 부하 분산을 수행한다.
- **자동 페일오버**
  MySQL 서버에 장애가 발생한 경우 자동으로 다른 MySQL 서버로 쿼리를 재시도하는데, 이때 지정된 부하 분산 방식에 따라 재시도할 MySQL 서버가 결정된다.


## 5. InnoDB 클러스터 구축

InnoDB 클러스터를 구축하는 것이 복잡해 보일 수 있지만, 실제로는 자동화된 기능을 통해 손쉽게 구축할 수 있다.

### 5.1) InnoDB 클러스터 요구사항

- InnoDB 클러스터를 구성하는 각 구성요소들은 다음 버전으로 설치돼야 한다.
    - MySQL 서버 5.7.17이상
    - MySQL 셸 1.0.8이상
    - MySQL 라우터 2.1.2이상
- InnoDB 클러스터의 MySQL 서버들은 모두 Performance 스키마가 활성돼 있어야 한다.
- MySQL 셸을 사용해 InnoDB 클러스터를 구성하기 위해 MySQL 셸이 설치될 서버에 파이썬 2.7 이상의 버전이 설치돼 있어야 한다.

### 5.2) InnoDB 클러스터 생성

- 예제를 위한 InnoDB 클러스터를 구성
    - MySQL 서버 8.0.22
    - MySQL 셸 8.0.22
    - MySQL 라우터 8.0.22

#### 사전 준비

서로 다른 서버 3대(ic-node1, ic-node2, ic-node3)에 MySQL이 설치돼 있고, 서로 다른 `sever_id`로 설정되어 있다고 가정하고 시작한다.

```bash
### MySQL 셸에 접속
docker exec -it ic-node1 mysqlsh

### 서버의 현재 설정이 클러스터에서 요구되는 사항을 충족하는지 확인
mysql|JS> dba.configureInstance("root@localhost:3306")
Please provide the password for 'root@localhost:3306': ************
Save password for 'root@localhost:3306'? [Y]es/[N]o/Ne[v]er (default No): n
Configuring local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as 6c7ed2c91482:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

applierWorkerThreads will be set to the default value of 4.

The instance '6c7ed2c91482:3306' is valid to be used in an InnoDB cluster.
The instance '6c7ed2c91482:3306' is already ready to be used in an InnoDB cluster.

Successfully enabled parallel appliers.
```

- 원격에서도 접속 가능한 새로운 DB 계정을 만들 것을 권장한다.
- 설정 변경이 필요한 옵션들은 `SET PERSIST` 또는 `SET PERSIST_ONLY`로 수정한 뒤 MySQL 서버를 재시작 한다.

  ```sql
  mysql|SQL> SELECT * FROM performance_schema.persisted_variables;
  +----------------------------------------+----------------+
  | VARIABLE_NAME                          | VARIABLE_VALUE |
  +----------------------------------------+----------------+
  | binlog_transaction_dependency_tracking | WRITESET       |
  | replica_parallel_workers               | 4              |
  | enforce_gtid_consistency               | ON             |
  | slave_parallel_workers                 | 4              |
  | server_id                              | 2812724118     |
  | gtid_mode                              | ON             |
  +----------------------------------------+----------------+
  ```
  ```bash
  ### 설정 값 수정
  mysql|SQL> SET PERSIST GTID_MODE = ON;
  mysql|SQL> SET PERSIST ENFORCE_GTID_CONSISTENCY = ON;
  mysql|SQL> SET PERSIST SERVER_ID = 1;
  mysql|SQL> SET GLOBAL super_read_only = OFF;
  ```

#### InnoDB 클러스터 인스턴스 생성

##### 클러스터 생성

- `dba.createCluster()` 메서드를 통해 클러스터를 생성한다.

  ```bash
  mysql|JS> var cluster = dba.createCluster("testCluster")
  A new InnoDB Cluster will be created on instance '6c7ed2c91482:3306'.

  Validating instance configuration at /var%2Frun%2Fmysqld%2Fmysqld.sock...

  This instance reports its own address as 6c7ed2c91482:3306

  Instance configuration is suitable.
  NOTE: Group Replication will communicate with other members using '6c7ed2c91482:3306'. Use the localAddress option to override.

  Creating InnoDB Cluster 'testCluster' on '6c7ed2c91482:3306'...

  Adding Seed Instance...
  Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
  At least 3 instances are needed for the cluster to be able to withstand up to
  one server failure.
  ```

- InnoDB 클러스터를 구축하려면 그룹 복제 설정 등의 복잡한 작업이 필요한데, `dba.createCluster()` 메서드에서 모두 자동으로 처리해준다.

- `dba.createCluster()`에서 수행해주는 작업
    - InnoDB 클러스터에 대한 정보를 저장할 메타데이터 데이터베이스(mysql_innodb_cluster_metadata) 생성 및 메타데이터 설정
    - 그룹 복제 설정 및 시작
    - 그룹 복제 분산 복구에서 사용될 DB 계정 생성

- InnoDB 클러스터는 기본적으로 싱글 프라이머리 모드로 실행된다.
    - 멀티 프라이머리 모드로 클러스터를 생성하고 싶다면 옵견을 지정해서 클러스터를 생성하면 된다.

  ```bash
  mysql|JS> var cluster = dba.createCluster("testCluster", {multiPrimary: true})
  ```

##### 클러스터 조회

- `dba.getCluster()` 메서드로 클러스터 객체의 상태를 조회할 수 있다.

  ```bash
  mysql|JS> var cluster = dba.getCluster()
  mysql|JS> cluster.status()
  {
      "clusterName": "testCluster",
      "defaultReplicaSet": {
          "name": "default",
          "primary": "6c7ed2c91482:3306",
          "ssl": "REQUIRED",
          "status": "OK_NO_TOLERANCE",
          "statusText": "Cluster is NOT tolerant to any failures.",
          "topology": {
              "6c7ed2c91482:3306": {
                  "address": "6c7ed2c91482:3306",
                  "memberRole": "PRIMARY",
                  "mode": "R/W",
                  "readReplicas": {},
                  "replicationLag": "applier_queue_applied",
                  "role": "HA",
                  "status": "ONLINE",
                  "version": "8.0.32"
              }
          },
          "topologyMode": "Single-Primary"
      },
      "groupInformationSourceMember": "6c7ed2c91482:3306"
  }
  ```

    - status에 "OK_NO_TOLERANCE"는 클러스터가 노드 한 대로만 구성되어 장애에 안전하지 않다는 의미이다.

#### InnoDB 클러스터 인스턴스 추가

- 클러스터에 서버를 추가하려면 `<Cluster>.addInstance()` 메서드를 사용하면 된다.
    - `addInstance()` 메서드는 현재 클러스터 서버와 비교해서 추가될 서버의 데이터 동기화가 필요한지 판단한다.
    - 동기화가 필요한 경우, 그룹 복제의 분산 복구를 수행한다.

- 프라이머리 서버가 아니더라도 클러스터 내의 어떤 서버에서든 인스턴스 추가 작업을 진행할 수 있다.

```bash
docker exec -it ic-node1 mysqlsh

mysql|JS> \connect root@ic-node1:3306

mysql|ic-node1|JS> var cluster = dba.getCluster()
mysql|ic-node1|JS> cluster.addInstance("root@ic-node2:3306")
mysql|ic-node1|JS> cluster.addInstance("root@ic-node3:3306")

mysql|ic-node1|JS> cluster.status()
```

- 클러스터에 이미 프라이머리 서버가 존재하므로, 추가된 두 서버가 읽기 전용 세컨터리로 동작하고 있는 것을 확인할 수 있다.


#### MySQL 라우터 설정

- 별도의 라우터용 서버(mysql-router-server1)가 존재하고, MySQL 라우터가 설치되어 있다고 가정한다.

  ```bash
  router_linux> mysqlrouter --bootstrap root@ic-node1:3306 --name icrouter1 \ 
  --directory /tmp/myrouter --account icrouter --user root
  ```

- 지정한 경로에 디렉터리 및 파일이 생성된 것을 확인한다.

  ```planText
  router_linux> ls -l /tmp/myrouter
  total
  drwx------ data
  drwx------ log
  -rW------- mysqlrouter.conf
  -rw------- mysqlrouter. key
  drwx------ run
  -rWX------ start.sh
  -rWX------ stop.sh
  ```

- InnoDB 클러스터에서 부트스트랩으로 자동 생성된 라우터용 DB 계정과 라우터 서버 등록 내역을 확인한다.

  ```bash
  mysql|JS> \connect root@ic-node1:3306

  mysql|ic-node1|JS> \sql SELECT user, host FROM myusql.user WHERE user = 'root';

  mysql|ic-node1|JS> \sql SHOW GRANTS FOR 'root'@'%';

  mysql|ic-node1|JS> var cluster = dna.getCluster()
  mysql|ic-node1|JS> cluster.listRouters()
  ```

    - 부트스트랩이 완료되면 MySQL의 기본 프로토콜로 연결되는 읽기/읽기-쓰기 포트, X 프로토콜로 연결되는 읽기/읽기-쓰기 포트 총 4개의 TCP 포트를 사용하도록 설정된다.


##### 라우터 동작 방식

- MySQL 라우터는 내부적으로 플러그인 형태의 아키텍처로 구성된다.
    - 메타데이터 캐시 플러그인: 라우터에서 접속할 InnoDB 클러스터의 정보를 구성하고 관리
    - 커엑션 라우팅 플러그인: 애플리케이션 서버로부터 유입된 쿼리 요청을 InnoDB 클러스터로 전달

##### MySQL 라우터 옵션

- [metadata_cache] 관련 옵션
    - `ttl`
      내부적으로 캐싱하고 있는 클러스터 메타데이터를 갱신하는 주기
    - `use_gr_notifications`
      그룹 복제에서 발생하는 변경사항에 대해 라우터 알림을 받을 수 있고, 알림 수신 후 현재 캐싱하고 있는 클러스터 메타데이터를 갱신

- [routing] 관련 옵션
    - `destinations`
      쿼리 요청을 전달할 대상 서버를 지정
      MySQL 서버 IP를 지정하는 정적인 형태와 URI 포맷의 동적인 형태가 있으며, URI에서는 다음의 세 옵션을 사용할 수 있음
        - `role`: PRIMARY, SECONDARY, PRIMARY_AND_SECONDARY
        - `disconnect_on_promoted_to_primary`: 클러스터에서 세컨더리 서버가 프라이머리로 승격되었을 때 기존 클라이언트 연결을 종료할지 여부
        - `disconnect_on_metadata_unavailable`: 메타데이터 갱신이 불가능할 때 기존 클라이언트 연결을 모두 종료할지 여부
    - `routing_strategy`
      MySQL 라우터가 연결 대상 서버를 선택하는 방식을 제어

        - `round-robin`: 라운드로빈 방식으로 선택
        - `round-robin-with-fallback`: 세컨더리 서버에서 라운드로빈 방식으로 선택하고, 세컨더리 서버가 존재하지 않으면 프라이머리 서버에 대해 라운드로빈 방식으로 선택
        - `first-available`: 연결 가능한 목록의 첫 번째 서버에 연결하고, 에러가 발생하면 다음 목록의 서버에 연결
        - `next-available`: first-available 방식과 유사하지만, 에러가 발생한 서버에 대해 연결 불가로 표시하고 라우터가 재시작될 때까지 연결 대상에 포함되지 않음

- 라우터에 대한 부트스트랩이 완료되면 자동으로 실행되지 않으므로, 라우터를 수동으로 실행한다.

  ```bash
  router_linux> /tmp/myrouter/start.sh

  router_linux> netstat -tnlp | grep mysqlrouter
  ```

- 이제 클라이언트에서는 MySQL 라우터 서버를 통해 InnoDB 클러스터로 쿼리를 실행할 수 있다.
- 라우터를 통해 실행된 쿼리가 실제로 어떤 MySQL 서버에 전달된 것인지를 확인하려면 hostname과 port 시스템 변수를 조회하면 된다.

  ```bash
  ### 라우터의 기본 쓰기 포트(6446)로 연결해서 쿼리를 실행
  mysql|JS> \connect root@mysql-router-server1:6446
  mysql|mysql-router-server1:6446|JS> \sql SELECT @@host, @@port;

  ### 라우터의 기본 읽기 포트(6447)로 연결해서 쿼리를 실행
  mysql|JS> \connect root@mysql-router-server1:6447
  mysql|mysql-router-server1:6447|JS> \sql SELECT @@host, @@port;
  ```

    - 쓰기 포트로 연결해서 실행한 쿼리를 프라이머리로 전달되고, 읽기 포트로 연결해서 실행한 쿼리는 세컨더리 서버들 중 한 서버로 전달된다.
    - 현재 구성이 싱글 프라이머리 모드이므로 쓰기 포트는 프라이머리 서버와 1:1로 매핑되었지만, 멀티 프라이머리 모드에서는 쓰기 포트에 연결해서 실행한 쿼리도 다른 프라이머리 서버로 전달될 가능성이 있다.

