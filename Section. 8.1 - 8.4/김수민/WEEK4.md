# Chapter 8. 인덱스

MySQL 서버의 옵티마이저가 발전하고 성능이 개선되었다고 해도 인덱스에 대한 기본지식은 지금도 앞으로도 개발자나 관리자에게 매우 중요한 부분이며, 쿼리 튜닝의 기본이 될 것이다.

## 8.1 디스크 읽기 방식

데이터베이스 성능 튜닝은 어떻게 디스크 IO를 줄이느냐가 관건일 때가 상당히 많다.

### **8.1.1 HDD (Hard Disk Driver)와 SSD (Solid State Drive)**

CPU, 메모리 등 주요 장치 → **전자식** 장치

**HDD** → **기계식** 장치, 병목지점

**SSD** → 기계식 HDD를 대체하기 위한 **전자식 저장매체**, **플래시 메모리**를 장착하여 디스크 원판 회전이 필요 없음

![image](https://github.com/user-attachments/assets/3f2bc951-aba8-4022-900b-623e6c552c07)


디스크 헤더를 움직이지 않고 한 번에 많은 데이터를 읽는 순차 IO에서는 SSD가 HDD보다 약간 빠르거나 비슷한 성능을 보인다. 하지만 랜덤 IO는 SSD가 훨씬 빠르다.

DB 서버에는 랜덤 IO 작업이 대부분이므로 SSD의 장점은 DBMS용 스토리지에 최적이다. 일반적인 웹 서비스 (OLTP*) 환경에서는 SSD가 HDD보다 훨씬 빠르다.

.* **OLTP (Online Transactional Processing)**: 온라인 뱅킹, 쇼핑, 주문 입력 또는 텍스트 메시지 전송 등 **동시에 발생하는 다수의 트랜잭션을 실행하는 데이터 처리 유형**이다. 이러한 트랜잭션은 전통적으로 경제 또는 재무 트랜잭션이라고 칭한다.

### **8.1.2 랜덤 IO와 순차 IO**

디스크에 데이터를읽고 쓰는데 걸리는 시간은 디스크 헤더를 읽고 쓸 위치로 옮기는 단계에서 결정된다. 즉, 디스크의 성능은 디스크 헤더의 위치 이동 없이 얼마나 많은 데이터를 한 번에 기록하느냐에 의해 결정된다고 볼 수 있다. 그래서 여러 번 읽기 또는 쓰기 요청을 하는 랜덤 IO 작업이 부하가 훨씬 크다.

디스크 원판이 없는 SSD에서도 랜덤 IO는 순차 IO보다 전체 Throughput*이 떨어진다.

**랜덤 IO나 순차 IO 모두 파일에 쓰기를 실행하면 반드시 동기화 (fsync 또는 flush 작업)가 필요하다.** 순차 IO인 경우에도 이런한 파일 동기화 작업이 빈번히 발생한다면 랜덤 IO와 같이 비효율적인 형태로 처리될 때가 많다.

일반적으로 쿼리를 튜닝하는 것은 쿼리를 처리하는 데 꼭 필요한 데이터만 읽로고 하여 랜덤 IO 자체를 줄여주는 것이 목적이라고 할 수 있다.

**데이터를 읽기 위해 인덱스 range 스캔은 주로 랜덤IO를, 풀 테이블 스캔은 순차 IO를 사용한다.** 그래서 큰 데이틀의 레코드 대부분을 읽는 작업에선느 인덱스를 자용하지 않고 풀 테이블 스캔을 사용하도록 유도할 때도 있다. 이런 형태는 OLTP 성격의 웹 서비스보다는 데이터 웨어하우스나 통계 작업에서 자주 사용된다.

.* Throughput: 처리율, 단위시간 당 전송량.

## 8.2 인덱스란?

DBMS가 DB 테이블의 모든 데이터를 검색해서 원하는 결과를 가져오려면 시간이 오래걸니다. 그래서 **칼럼의 값과 해당 레코드가 저장된 주소를 key-value 쌍으로 하여 인덱스**를 만들어 두는 것이다. 인덱스는 칼럼의 값을 주어진 순서로 **미리 정렬**해서 보관된다.

인덱스가 많은 테이블은 INSERT, UPDATE, DELETE 문장의 처리가 느려지지만, 정렬된 인덱스를 가지고 있으므로 SELECT 문장은 매우 빠르게 처리할 수 있다. 테이블의 인덱스를 하나 더 추가할지 말지는 데이터의 저장 속돌르 어디까지 희생할 수 있는지, 읽기 속도를 얼마나 더 빠르게 만들어야 하느냐에 따라 결정해야 한다.

- **인덱스의 역할별 구분**
    - **프라이머리 키 (Primary Key)**
        
        : 레코드를 식별하는 기준값. NULL과 중복을 허용하지 않는다.
        
    - **세컨더리 인덱스 (Secondary Key)**
        
        : PK를 제외한 나머지 모든 인덱스. 유니크 인덱스는 PK와 성격이 비슷하고 PK를 대체해서 사용할 수 있다고 해서 대체 키라고도 불린다. 별도로 분류하기도 하고 세컨더리 인덱스로 분류하기도 한다.
        
- **데이터 저장 방식 (알고리즘)별 구분**
    - **B-Tree 인덱스**
        
        : 가장 일반적이고, 오래되어 성숙한 알고리즘이다. 칼럼의 값을 변형하지 않고 원래의 값을 이용해 인덱싱한다. R-Tree 인덱스 알고리즘은 B-Tree의 응용 알고리즘이다.
        
    - **Hash 인덱스**
        
        : 칼럼의 값으로 해시값을 계산해서 인덱싱한다. 매우 빠른 검색을 지원한다. 하지만 값을 변형해서 인덱승하므로 Prefix 일치와 같이 일부만 검색하거나 범위를 검색할 때는 사용할 수 없다. 주로 인메모리 DB에서 많이 사용한다.
        
    - **Fractal-Tree 인덱스, Merge-Tree 인덱스**
        
        : 최근에 이용되는 알고리즘. Fractal-Tree Index는 B-Tree 변형 구조로 쓰기 성능을 높이기 위해 버퍼를 활용하며, Merge-Tree Index는 LSM-Tree 계열로 쓰기 후 배치 병합을 통해 성능을 최적화하는 방식이다.
        

인덱스가 유니크한지 아닌지는 쿼리 옵티마이저에게 상당히 중요한 문제가 된다. **유니크 인덱스에 대해 동등 조건 (Equal, =)으로 검색한다는 것은 항상 1건의 레코드를 찾으면 더 찾지 않아도 된다는 것을 옵티마이저에게 알려주는 효과를 낸다.** → 자세한 것은 인덱스와 쿼리 실행 계획을 살펴보며 배울 것.

## 8.3 B-Tree 인덱스

인덱싱 알고리즘 중 가장 일반적으로 사용되고, 가장 먼저 도입되었고, 아직도 가장 범용적인 목적으로 사용되는 알고리즘. 전문 검색 같은 특수 요건을 제외하고 **대부분의 일반적인 인덱스 용도에 적합.**

B-Tree의 ‘B’는 Binary가 아니라 **Balanced**이다.

칼럼의 원래 값을 변형시키지 않고 인덱스 구조체 내에서는 항상 정렬된 상태로 유지한다.

### **8.3.1 구조 및 특성**

![image](https://github.com/user-attachments/assets/f39ea6f8-7167-4b9c-a0cd-fcf8a6d62f33)

최상위 **루트노드**, 가장 하위의 **리프 노드**, 중간의 **브랜치 노드**로 구성된다.

데이터 파일의 **레코드는 INSERT된 순서로 저장되지 않고 임의의 순서로 저장**되어 있다. 테이블 레코드의 변경이나 삭제가 없다면 순차적으로 저장되지만, 테이블의 레코드가 삭제되어 빈 공간이 생기면 그 다음 **INSERT는 가능한 삭제된 공간을 재활용**하도록 DBMS가 설계된다.

```sql
대부분 RDBMS의 데이터 파일에서 레코드는 특정 기준으로 정렬되지 않고 임의의 순서로 저장된다. 하지만 InnoDB 테이블에서 레코드는 클러스터되어 디스크에 저장되므로 기본적으로 PK 순서로 정렬된다. 다른 DBMS에서는 클러스터링 기능이 선택사항이지만, InnoDB에서는 사용자가 별도의 옵션을 선택하지 않아도 디폴트로 클러스터링 테이블이 생성된다.
```

InnoDB 스토리지 엔진을 사용하는 테이블에서는 PK가 RowID 역할을 한다.

**MyISAM 테이블은 세컨더리 인덱스가 물리적인 주소를 가지는 반면 InnoDB 테이블은 PK를 주소처럼 사용하기 때문에 논리적인 주소를 가진다고 볼 수 있다.**

그래서 InnoDB 테이블에서 인덱스를 통해 레코드를 읽을 때는 인덱스에 저장돼 있는 PK 값을 이용해 PK 인덱스를 한 번 더 검색 한 후, PK 인덱스의 리프 레이지에 저장돼 있는 레코드를 읽는다. 즉, InnoDB  스토리지 엔진에서는 **모든 세컨더리 인덱스 검색에서 PK를 저장하고 있는 B-Tree를 한 번 더 검색하는 과정**이 필요하다.

### **8.3.2 B-Tree 인덱스 키 추가 및 삭제**

- **인덱스 키 추가**
    
    새로운 키 값이 B-Tree에 저장될 때 테이블의 스토리지 엔진에 따라 새로운 키 값이 즉시 인덱스에 저장될 수도 있고 그렇지 않을 수도 있다.
    
    ```sql
    B-Tree에 저장될 때는 키 값을 이용해 B-Tree상의 적절한 위치를 검색해야 한다. 저장될 위치가 결정되면 레코드의 키 값과 대상 레코드의 주소 정보를 B-Tree의 리프노드에 저장한다. 리프노드가 꽉차서 더는 저장할 수 없을 때는 리프노드가 분리 (Split)돼야 하는데, 이는 상위 브랜치 노드까지 처리가 필요하다. 이런 작업 탓에 B-Tree는 상대적으로 새로운 키를 추가하는 작업에 비용이 많이 드는 것으로 알려져 있다.
    ```
    
    인덱스 추가로 INSERT, UPDATE 문장이 어떤 영향을 받는지는 테이블의 칼럼 수, 칼럼의 크기, 인덱스 칼럼의 특성 등을 알아야 한다. 애략적으로 레코드 추가 비용을 1, 인덱스에 키를 추가하는 작업 비용을 1.5정도로 예측할 수 있다.
    
    중요한 것은 **이 비용의 대부분이** 메모리, CPU에서 처리하는 시간이 아니라, **디스크로부터 인덱스 페이지를 읽고 쓰는데 걸리는 시간**이라는 것이다.
    
    ```sql
    MyISAM이나 MEMORY 스토리지 엔진을 사용하는 테이블에서는 INSERT문이 실행되면 즉시 새로운 값을 B-Tree 인덱스에 변경한다.
    하지만 InnoDB 스토리지 엔진은 중복체크가 필요한 PK, 유니크 키 등을 제외하고는 필요하면 인덱스 키 추가 작업을 지연시켜 나중에 처리할 수 있다. -> 4.2.10 체인지 버퍼 참조
    ```
    
- **인덱스 키 삭제**
    
    B-Tree 키 값이 삭제되는 경우, 해당 키 값이 저장된 B-Tree의 리프 노드를 찾아 **삭제 마크**만 하면 작업이 완료된다.
    
    삭제 마킹된 인덱스 키 공간은 그대로 방치하거나 재활용할 수 있다. 인덱스 키 삭제로 인한 마킹 작업 또한 디스크 쓰기가 필요한 디스크 IO 작업이다.
    
- **인덱스 키 변경**
    
    B-Tree의 키 값 변경 작업은 먼저 **기존 인덱스 키 값을 삭제한 후, 다시 새로운 키 값을 추가**하는 형태로 처리된다.
    
    인덱스의 키 값은 그 값에 따라 저장될 리프 노드의 위치가 결정되므로 B-Tree의 키 값이 변경되는 경우 단순히 인덱스상의 키 값만 변경하는 것은 불가능하다.
    
- **인덱스 키 검색**
    
    인덱스 관리에 따르는 추가 비용을 감수하면서 인덱스를 구축하는 이유는 빠른 검색을 위해서이다.
    
    **B-Tree 인덱스를 이용한 검색은 100% 일치 또는 인덱스 키 값의 앞부분만 일치하는 경우에 사용할 수 있다. 부등호 비교 조건에서도 사용할 수 있다.**
    
    **인덱스를 구성하는 키 값의 뒷부분만 검색하거나, 인덱스의 키 값에 변형이 가해진 경우는 인덱스를 사용할 수 없다.** 따라서 함수나 연산을 수행한 결과로 정렬한다거나 검색하는 작업은 B-Tree의 장점을 이용할 수 없다.
    
    ```sql
    InnoDB 테이블에서 지원하는 레코드 잠금이나 넥스트 키 락 (갭 락)은 검색을 수행한 인덱스를 잠근 후 테이블의 레코드를 잠그는 방식으로 구현되어 있다.
    따라서 UPDATE,DELETE 수행 시 적절한 인덱스가 없으면 불필요하게 많은 레코드를 잠그거나, 심지어 테이블의 모든 레코드를 잠글 수도 있다. InnoDB 스토리지 엔진에서는 그만큼 인덱스의 설계가 중요하다.
    ```
    

### **8.3.3 B-Tree 인덱스 사용에 영향을 미치는 요소**

B-Tree 인덱스는 인덱스를 구성하는 칼럼의 크기와 레코드의 건수, 그리고 유니크한 인덱스 키 값의 개수 등에 의해 검색이나 변경 작업의 성능이 영향을 받는다.

- **인덱스 키 값의 크기**
    
    InnoDB 스토리지 엔진에서 **디스크에 데이터를 읽고 쓰는 가장 기본 단위**를 **페이지 (Page)** 또는 **블록 (Block)**이라고 한다.인덱스도 결국 페이지 단위로 관리된다.
    
    일반적으로 **DBMS의 B-Tree는 자식 노드의 개수가 가변적인 구조**다. **자식 노드의 수는 인덱스의 페이지 크기와 키 값의 크기에 따라 결정된다.**
    
    ```sql
    MySQL 5.7부터 `innodb_page_size` 시스템 변수를 이용해 InnoDB 스토리지 엔진의 페이지 크기를 4KB ~ 64KB사이의 값으로 선택할 수 있고, 기본 값은 16KB이다.
    인덱스 키가 16바이트라고 가정하고, 자식 노드 주소가 대략 6바이트 ~ 12바이트로 구성되므로 한 페이지에는 약 16*1024/(16+12) = 585개의 인덱스를 저장할 수 있다. 즉, 585개의 자식 노드를 가질 수 있는 B-Tree가 된다. 
    ```
    
    **인덱스를 구성하는 키 값의 크기가 커지면 한 페이지에 들어가는 레코드 수가 적어지므로 디스크로부터 읽어야 하는 횟수가 늘어나고 그만큼 느려진다.** 또, 인덱스 키 값의 길이가 길어진다는 것은 전체적인 인덱스 크기가 커진다는 것을 의미한다. 인덱스를 캐시해두는 InnoDB 버퍼 풀이나 MyISAM 캐시 영역은 크기가 제한적이고, **하나의 레코드를 위한 인덱스 크기가 커질 수록 메모리에 캐시해둘 수 있는 레코드 수는 줄어든다.** 그러면 자연히 메모리 효율이 떨어지는 결과를 가져온다.
    
- **B-Tree 깊이 (Depth)**
    
    B-Tree 인덱스의 깊이는 상당히 중요하지만 직접 제어할 방법은 없다.
    
    인덱스 키 값의 크기가 처지면 커질 수록 하나의 인덱스 페이지가 담을 수 있는 인덱스 키 값의 개수가 적어지고, 그 때문에 같은 레코드 건수라 하더라도 B-Tree의 깊이가 깊어져서 디스크 읽기가 더 많이 필요하게 된다.
    
    **인덱스 키 값의 크기는 가능하면 작게 만드는 것이 좋다.**
    
    실제로는 아무리 대용량 DB라도 B-Tree 깊이가 5단계 이상까지 깊어지는 경우는 흔치 않다.
    
- **선택도 (=Selectivity, 기수성=Cardinality)**
    
    **모든 인덱스 키 값 가운데 유니크한 값의 수**를 의미한다. 인덱스에서 선택도와 기수성은 거의 같은 의미로 사용된다.
    
    인덱스 키 값 가운데 중복된 값이 많으면 많을 수록 기수성과 선택도는 떨어진다. **인덱스는 선택도가 높을 수록 검색 대상이 줄어들기 때문에 그만큼 빠르게 처리된다.**
    
    즉, 인덱스에서 유니크한 값의 개수는 인덱스나 쿼리의 효율성에 큰 영향을 미친다.
    
    ```sql
    선택도가 좋지 않더라도 정렬, 그루핑 같은 작업을 위한 인덱스를 만드는 것이 훨씬 나은 경우도 많다. 인덱스가 항상 검색에만 사용되는 것은 아니므로 여러가지 용도를 고려해 적절히 인덱스를 설계해야 한다.
    ```
    
- **읽어야 하는 레코드의 건수**
    
    인덱스를 통해 테이블의 레코드를 읽는 것은 인덱스를 거치지 않고 바로 테이블의 레코드를 읽는 것보다 높은 비용이 드는 작업이다.
    
    일반적인 DBMS의 옵티마이저에서는 인덱스를 통해 레코드 1건을 읽는 것이 테이블에서 직접 레코드 1건을 읽는 것보다 4~5배정도 비용이 더 많이 드는 작업으로 예측한다. (RDBMS 서버별로, MySQL 서버에서는 코스트 모델* 설정 따라 달라질 수 있음.)
    
    즉, **인덱스를 통해 읽어야 할 레코드의 건수가 전체 테이블 레코드의 20~25%를 넘어서면** 인덱스를 이용하지 않고 **테이블을 모두 읽어서** 필요한 레코드만 가려내는 **필터링 방식으로 처리하는 것이 효율적**이다.
    
    .*코스트 모델: 10.1.3절 코스트 모델 (Cost Model) 참조
    

### **8.3.4 B-Tree 인덱스를 통한 데이터 읽기**

어떤 경우에 인덱스를 사용하도록 유도할지, 또는 사용하지 못하도록 할지 판단하려면 MySQL의 각 스토리지 엔진이 어떻게 인덱스를 이용 (경유)해서 실제 레코드를 읽어내는지 알아야 한다. MySQL은 대표적으로 다음 세가지 방법을 사용한다.

- **인덱스 레인지 스캔**
    
    ![image](https://github.com/user-attachments/assets/4537adbd-8378-4c45-a7ee-3acd61471552)

    
    인덱스 접슨 방법 중 가장 대표적인 접근 방식으로, 뒤의 두 가지 접근 방식보다 빠른 방법이다.
    
    검색해야 할 인덱스의 범위가 결정됐을 때 사용하는 방식이다. 검색하려는 값의 수나 검색 결과 레코드 건수와 관계 없이 레인지 스캔이라고 표현한다.
    
    ```sql
    루트 노드에서부터 비교를 시작해 브랜치 노드를 거치고 최종적으로 리프 노드까지 찾아들어가야만 필요한 레코드의 시작점을 찾을 수 있다. 시작할 위치를 찾으면 그때부터 리프 노드의 레코드만 순서대로 읽으면 된다. 스캔을 멈춰야 할 위치에 다다르면 지금까지 읽은 레코드를 반환하고 쿼리를 끝낸다.
    
    위 사진은 인덱스만 읽어나가는 과정이다.
    
    인덱스 레인지 스캔은 다음과 같이 크게 3단계를 거친다.
    1. **인덱스 탐색 (Index seek)**: 인덱스에서 조건을 만족하는 값이 저장된 위치를 찾는다.
    2. **인덱스 스캔 (Index scan)**: 1번에서 탐색된 위치부터 필요한 만큼 인덱스를 차례로 쭉 읽는다. (1번과 2번을 합쳐 인덱스 스캔으로 통칭하기도 한다.)
    3. 2번에서 읽어들인 인덱스 키와 레코드 주소를 이용해 레코드가 저장된 페이지를 가져오고, **최종 레코드**를 읽어온다. (**커버링 인덱스***로 처리되는 쿼리는 이 단계 필요 없음.)
    ```
    
    .* 커버링 인덱스 (Covering Index): 쿼리에서 요청한 모든 컬럼이 인덱스에 포함되어 있어, 테이블을 조회하지 않고 인덱스만으로 결과를 반환할 수 있는 인덱스
    
- **인덱스 풀 스캔**
    
    ![image](https://github.com/user-attachments/assets/703ee34c-bbdd-481e-bf03-239fbcc58f91)

    
    **인덱스의 처음부터 끝까지 모두 읽는 방식**. 일반적으로 인덱스의 크기는 테이블 크기보다 작고, 디스크 IO를 줄일 수 있어 **테이블 풀스캔보다 효율적**이다.
    
    인덱스를 이용하지만 효율적인 방식은 아니며, 일반적으로 인덱스를 생성하는 목적은 아니다. 인덱스 뿐만 아니라 데이터 레코드까지 모두 읽어야 한다면 절대 이 방식으로 처리되지 않는다.
    
    대표적으로, 쿼리의 조건절에 사용된 칼럼이 인덱스의 첫번째 칼럼이 아닌 경우 사용된다.
    
    ex) 인덱스는 A, B, C 칼럼 순서로 만들어져 있지만, 쿼리 조건절은 B칼럼이나 C칼럼으로 검색
    
- **루스 (Loose) 인덱스 스캔**
    
    ![image](https://github.com/user-attachments/assets/786f4f43-f737-4373-8dba-ce2a4143a4cc)

    
    오라클과같은 DBMS의 인덱스 스킵 스캔과 작동방식이 비슷 하다.
    
    인덱스 레인지 스캔과 비슷하게 작동하지만 **중간에 필요하지 않은 인덱스 키 값은 무시하고 다음으로 넘어가는 형태**로 처리한다. 루스 인덱스 스캔을 사용하려면 여러 가지 조건을 만족해야 한다. (10장 실행 계획 참조)
    
    일반적으로 GROUP BY 또는 MAX(), MIN() 함수에 대해 최적화를 하는 경우 사용된다.
    
    앞에 나온 두 방법 (**인덱스 레인지 스캔, 인덱스 풀 스캔**)은 루스 인덱스 스캔과는 반대로 **타이트 (tight)인덱스 스캔으로 분류**한다.
    
- **인덱스 스킵 스캔**
    
    ![image](https://github.com/user-attachments/assets/4f948762-8816-45f4-862d-72190f0aa328)

    
    복합 인덱스에서 **선두 컬럼 조건 없이 후속 컬럼 조건으로 여러 번 인덱스를 부분 스캔**하는 방식. MySQL 8.0 버전부터 도입되었다.
    
    루스 인덱스 스캔은 GROUP BY 작업을 처리하기 위해 인덱스를 사용하는 경우에만 적용할 수 있는 반면, 인덱스 스킵 스캔은 WHERE 조건절의 검색을 위해 사용 가능하도록 용도가 훨씬 넓어졌다.
    
    현재 인덱스 스킵 스캔은 다은과 같은 제약이 있다.
    
    1. WHERE 조건절에 조건이 없는 인덱스의 유니크한 값의 개수가 적어야 한다.
        
        → 쿼리 실행 계획에서 유니크한 값의 개수가 매우 많다면 MySQL 옵티마이저는 인덱스에서 스캔해야 할 시작 지점을 검색하는 작업이 많이 필요해진다.
        
    2. 쿼리가 인덱스에 존재하는 칼럼만으로 처리 가능해야 한다. (커버링 인덱스)
        
        → 인덱스만으로 결과를 반환해야 테이블 접근 없이 스킵 스캔의 성능 이점을 누릴 수 있으며, 그렇지 않으면 테이블 접근이 반복되어 오히려 비효율적일 수 있다.
        
    

### **8.3.5 다중 칼럼 (Multi-cloumn) 인덱스 (= 복합 칼럼 인덱스)**

![image](https://github.com/user-attachments/assets/a02dd3c9-134f-457e-87d6-304c57e0428b)


두 개 이상의 칼럼으로 구성된 인덱스.

**두 번째 칼럼은 첫 번째 칼럼에 의존해서 정렬되어 있다.** 즉, 두 번째 칼럼의 정렬은 첫 번째 칼럼이 똑같은 레코드에서만 의미가 있으므로 **선두 칼럼부터 순서대로 조건이 주어질 때만 인덱스를 효율적으로 활용**할 수 있다. 그렇기 때문에 **다중 칼럼 인덱스에서는 인덱스 내에서 각 칼럼의 순서가 중요**하다.

### **8.3.6 B-Tree 인덱스의 정렬 및 스캔 방향**

인덱스를 생성할 때 설정한 정렬 규칙에 따라서 인덱스의 키 값은 항상 오름차순이거나 내림차순으로 정렬되어 저장된다. 하지만 인덱스를 어느 방향으로 읽을지는 쿼리에 따라 옵티마이저가 실시간으로 만들어내는 실행 계획에 따라 결정된다.

- **인덱스의 정렬**
    
    MySQL 8.0 버전부터는 칼럼 단위로 정렬 순서를 혼합해서 인덱스를 생성할 수 있게 되었다.
    
    ```sql
    CREATE INDEX idx_team_name_userscore ON employees (team_name ASC, user_score DESC)
    ```
    
- **인덱스 스캔 방향**
    
    인덱스는 생성 시점에 오름차순 또는 내림차순 정렬이 결정되지만 쿼리가 그 인덱스를 사용하는 시점에 **인덱스를 읽는 방향에 따라 오름차순 또는 내림차순 정렬 효과**를 얻을 수 있다.
    
    즉, 인덱스가 실제 오름차순인지 내림차순인지와 관계 없이 인덱스를 읽는 순서만 변경해서 정렬 효과를 얻는다.
    
- **내림차순 인덱스**
    
    인덱스 정렬 방향: B-Tree 좌 → 우 (오름차순 → 값이 작은 인덱스부터, 내림차순 → 큰 값의 인덱스부터)
    
    스캔 방향: 정순 (좌 → 우), 역순 (우 → 좌)
    
    InnoDB 스토리지 엔진에서 정순 스캔과 역순 스캔은 페이지 간의 double linked list를 통해 전진 (forward) 하는지 후진 (backward) 하는지의 차이이지만, 실제 내부적으로는 InnoDB에서 인덱스 역순 스캔이 정순 스캔에 비해 느릴 수에 없다.
    
    ![image](https://github.com/user-attachments/assets/58e685ec-3680-41c0-9ae4-5ade6598218d)

    
    그 이유는 페이지 락이 forward index scan에 적합한 구조이며, 위 그림에서 볼 수 있듯 페이지 내에서 인덱스 레코드가 단방향으로만 연결된 구조이기 때문이다.
    

### **8.3.7 B-Tree 인덱스의 가용성화 효율성**

쿼리의 WHERE조건이나 GROUP BY, ORDER BY절이 어떤 경우에 인덱스를 사용할 수 있고 어떤방식으로 사용할 수 있는지 식별할 수 있어야 한다. 그래야 쿼리 조건을 최적화하거나, 역으로 쿼리에 맞게 인덱스를 최적으로 생성할 수 있다.

- **비교 조건의 종류와 효율성**
    
    다중 칼럼 인덱스에서 각 칼럼의 순서와 그 칼럼에 사용된 조건이 동등비교 (=)인지 비교 (>, <)의 범위 조건인지에 따라 각 인덱스 칼럼의 활용 형태와 효율이 달라진다.
    
    ex)
    
    ```sql
    SELECT * FROM dept_emp
    WHERE dept_no='d002' AND emp_no >= 10114;
    
    case1) INDEX (dept_no, emp_no) -> dept_no='d002' AND emp_no >= 10114
    case2) INDEX (emp_no, dept_no) -> emp_no >= 10114 AND dept_no='d002'
    ```
    
    ![image](https://github.com/user-attachments/assets/6fbbb663-d4fc-474e-991b-0d044a755c82)

    
    case1) dept_no, emp_no 기준 정렬 → d002 찾음. → d002인동안 ≥ 10144인 레코드들 찾음.
    
    case2) emp_no, dept_no 기준 정렬 → ≥ 10144 찾음. → d002인 칼럼 다시 필터링 필요.
    
    공식 명칭은 아니지만 case1 인덱스에서의 두 조건처럼 작업의 범위를 결정하는 조건을 ‘**작업 범위 결정 조건**’이라고 하고, case2 인덱스의 두 조건처럼 비교 작업의 범위를 줄이지 못하고 단순히 거름종이 역할만 하는 조건을 ‘**필터링 조건**’ 또는 ‘**체크 조건**’이라고 표현한다.
    
- **인덱스의 가용성**
    
    B-Tree 인덱스의 특징은 **왼쪽 값을 기준 (Left-most)으로 오른쪽 값이 정렬**돼 있다는 것이다. 이는 하나의 칼럼 내에서 뿐만 아니라 다중 칼럼 인덱스의 칼럼에 대해서도 함께 적용된다.
    
    ![image](https://github.com/user-attachments/assets/af1457da-7167-442a-89a6-f4a76ac1a0ed)

    
    인덱스 키 값의 정렬 특성은 빠른 검색의 전제조건이다. 하나의 칼럼으로 검색해도 값의 왼쪽 부분이 없으면 인덱스 레인지 스캔 방식으로 검색이 불가능하다. 또, 다중 칼럼 인덱스에서도 왼쪽 칼럼의 값을 모르면 인덱스 레인지 스캔을 사용할 수 없다.
    
    즉, **선행 칼럼의 조건 없이 다른 칼럼의 조건으로만 검색한다면 인덱스를 효율적으로 사용할 수 없다.**
    
    인덱스의 왼쪽 값 기준 규칙은 GROUP BY절이나 ORDER BY절에도 똑같이 적용된다. → 나중에 자세히
    
- **가용성과 효율성 판단**
    
    기본적으로 B-Tree 인덱스의 특성상 다음의 조건에서는 인덱스를 작업 범위 결정 조건으로 사용할 수 없다. 경우에 따라 체크 조건으로는 인덱스를 사용할 수 있다.
    
    - **NON-EQUAL**로 비교된 경우
    - LIKE ‘%XX’, ‘_XX’, ‘%X%’ (앞부분이 아닌 **뒷부분 일치**) 형태로 문자열 패턴이 비교된 경우
    - 스토어드 함수나 다른 **연산자로 인덱스 칼럼이 변형**된 후 비교된 경우
        
        ex) `WHERE SUBSTRING(column, 1, 1) = ‘X’`
        
    - NOT-DETERMINISTIC 속성의 **스토어드 함수**가 비교 조건에 사용된 경우
        
        ex) `WHERE column = deterministic_function()`
        
    - **데이터 타입이 서로 다른 비교** (인덱스 칼럼의 타입을 변환해야 비교가 가능한 경우)
        
        ex) `WHERE char_column = 10`
        
    - 문자열 데이터 타입의 **콜레이션이 다른 경우**
        
        ex) `WHERE utf8_bin_char_column = euckr_bin_char_column`
        
    
    다른 일반적은 DBMS에서는 NULL 값이 인덱스에 저장되지 않지만 MySQL에서는 NULL 값도 인덱스에 저장되기 때문에 `WHERE column IS NULL`과 같은 조건도 작업 범위 결정 조건으로 인덱스를 사용한다.
    
    ex) INDEX ix_test (col1, col2, col3. …, coln)
    
    - **작업 범위 결정 조건으로 인덱스를 사용하지 못하는 경우**
        - col1 칼럼에 대한 조건이 없는 경우
        - col1 칼럼의 비교 조건이 위의 인덱스 사용 불가 조건 중 하나인 경우
        
        → 즉, **선행 칼럼에 대한 조건이 없거나 선행칼럼의 인덱스를 활용하지 못할 때**
        
    - **작업 범위 결정 조건으로 인덱스를 사용하는 경우 (2 < i < n인 임의 값)**
        - col1 ~ col(i-1) 칼럼까지 **동등 비교** 형태로 비교
        - coli 칼럼에 대해 **동등비교, 크다 작다, LIKE로 죄측 일치 패턴** 중 하나로 비교한 경우
        
        → 위 두 조건을 만족하는 쿼리는 col1~coli까지는 작업 범위 결정 조건으로 사용되고, col(i+1) ~ coln까지 조건은 체크 조건으로 사용된다.
        
    

## 8.4 R-Tree 인덱스

MySQL의 **공간인덱스 (Spitial Index)**는 **R-Tree 인덱스** 알고리즘을 이용해 **2차원의 데이터를 인덱싱하고 검색**하는 목적의 인덱스이다.

기본적인 내부 메커니즘은 B-Tree와 흡사하다. B-Tree 인덱스를 구성하는 인덱스 칼럼의 값이 1차원 스칼라 값인 반면, R-Tree 인덱스는 2차원 공간 개념 값이라는 차이가 있다.

MySQL의 Spatial Extension에는 크게 다음의 세 가지 기능이 포함되어 있다.

1) 공간 데이터를 저장할 수 있는 타입

2) 공간 데이터의 검색을 위한 공간 인덱스 (R-Tree 알고리즘)

3) 공간 데이터의 연산 함수 (거리 또는 포함 관계의 처리)

더 자세한 내용은 12.2절 공간검색 참고

### **8.4.1 구조 및 특성**

MySQL은 공간 정보의 저장 및 검색을 위해 여러가지 기하학적 도형 (Geometry) 정보를 관리할 수 있는 데이터 타입을 제공한다.

![image](https://github.com/user-attachments/assets/31cf9046-8028-4c7e-ba4e-874c06babecf)


Geometry 타입은 나머지 타입들의 Super타입이다.

R-Tree 알고리즘을 이해하기 위해서는 **MBR***이라는 개념을 알고 있어야 한다.

**이 사각형들의 포함 관계를 B-Tree 형태로 구현한 것이 R-Tre 인덱스**이다.

![image](https://github.com/user-attachments/assets/3c7b1a0c-4927-4e84-956b-6a716fb8e66b)


![image](https://github.com/user-attachments/assets/50bbc43e-d471-4f58-b13f-22e29a0de8d4)


최상위 MBR은 R-Tree의 루트노드에 저장되는 정보이며, 차상위 그룹 MBR은 R-Tree의 브랜치 노드, 최하위 레벨 MBR (각 도형을 둘러싼 가장 안쪽의 사각형, 각 도형 객체)은 리프노드에 저장된다.

.* **MBR (Minimum Bounding Rectangle): 해당 도형을 감싸는 최소 크기의 사각형**

### **8.4.2 R-Tree 인덱스의 용도**

**R-Tree는** **MBR 정보를 이용해 B-Tree 형태로 인덱스를 구축**한다.

Rectangle의 ‘R’과 B-Tree의 ‘Tree’를 섞어 R-Tree라는 이름이 불여졌으며, 공간 (Spatial) 인덱스라고도 한다.

일반적으로는 WG884(GPS) 기준의 위도, 경도 좌표 저장에 주로 사용된다.

![image](https://github.com/user-attachments/assets/b0d0ca7e-6080-4a4f-9786-7ff6a26dcbb4)


**R-Tree는 각 도형의 MBR의 포함 관계를 이용해 만들어진 인덱스**이다. 따라서 ST_Contains() 또는 ST_Within()등과 같은 포함 관계를 비교하는 함수로 검색을 수행하는 경우에만 인덱스를 이용할 수 있다.

현재 출시되는 MySQL 버전에서는 거리를 비교하는 ST_Distance(), ST_Distance_Sphere() 함수는 공간 인덱스를 효율적으로 사용하지 못하기 때문에 공간 인덱스를 사용할 수 있는 ST_Contains(), ST_Within()을 이용해 거리 기반 검색을 해야 한다.
