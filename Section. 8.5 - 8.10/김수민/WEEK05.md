## 8.5 전문 검색 (Full Text search)인덱스

문서 전체에 대한 분석과 검색을 위한 인덱싱 알고리즘 전체를 지칭한다.

문서의 내용 전체를 인덱스화해서 특정 키워드가 포함된 문서를 검색하는 전문 (Full Text) 검색에는 InnoDB나 MyISAM 스토리지 엔진에서 제공하는 일반적인 용도의 B-Tree 인덱스를 사용할 수 없다.

### 8.5.1 인덱스 알고리즘

전문 검색에서는 문서 본문의 키워드를 통해 빠른 검색을 할 수 있도록 **키워드로 인덱스를 구축**한다.

문서의 키워드를 인덱싱하는 기법에 따라 어근 분석, n-gram 알고리즘으로 구분할 수 있다. 구분자 (공백 또는 특정 기호 기준 토큰 분리)는 어근 분석과 n-gram 알고리즘에 포함된다.

MySQL 서버의 전문 검색 인덱스는 다음과 같은 과정을 거쳐 색인 작업이 수행된다.

1. **불용어 (Stop Word) 처리**
    
    : 검색에서 가치가 없는 단어 필터링 및 제거하는 작업. MySql에서는 불용어들을 테이블로 관리함.
    
2. **어근 분석 (Stemming)**
    
    : 검색어로 선정된 단어의 원형을 찾는 작업.
    

- **어근 분석 알고리즘**
    
    MySQL 서버에서는 **오픈소스 형태소 분석 라이브러리**인 **MeCab**을 플러그인 형태로 사용할 수 있도록 지원한다. 한국어, 일본어 등은 단어의 변형이 거의 없어 어근 분석보다는 형태소를 분석해 명사와 조사를 구분하는 것이 더 중요하다.
    
    문장 구조 인식을 위해 실제 언어의 샘플을 이용해 언어를 학습하는 과정이 필요하다.
    
    서구권 언어를 위한 형태소 분석기는 MongoDB에 사용되는 Snowball이라는 오픈소스가 있다.
    
- **n-gram 알고리즘**
    
    MeCap을 위한 형태소 분석은 매우 전문적인 알고리즘이기 때문에 전무적인 검색 엔진을 고려하는 것이 아니라면 범용적으로 적용하기 쉽지 않다. 이 단점을 보완하기 위해 n-gram 알고리즘이 도입되었다.
    
    **형태소 분석이 문장을 이해하는 알고리즘이라면, n-gram은 단순히 키워드를 검색해내기 위한 인덱싱 알고리즘이다.**
    
    n-gram이란 본문을 무조건 몇 글자씩 잘라서 인덱싱하는 방법이다.
    
    형태소 분석보다는 알고리즘이 단순하고 국가별 언어에 대한 이해와 준비 작업이 필요 없는 반면 만들어진 인덱스의 크기는 상당히 큰 편이다.
    
    ex) 2-gram 방식
    
    ![image (1)](https://github.com/user-attachments/assets/3f64f92e-421e-43cc-9acc-bf52e80f9c6f)

    
    ![image (2)](https://github.com/user-attachments/assets/86424d10-d58b-40e9-ae5d-3239d577f0ae)

    
    각 단어는 공백과 마침표를 기준으로 10개의 단어로 구분되고, 2글자씩 중첩해서 토큰으로 분리된다.
    
    MySQL 서버는 이렇게 생성된 토큰들에 대해 불용어를 걸러내는 작업을 수행한다. 불용어와 동일하거나 불용어를 포함하는 경우 걸러서 버린다. 이렇게 구분된 토큰을 단순한 B-Tree에 저장한다.
    

- **불용어 변경 및 삭제**
    
    불용어가 포함된 단어들도 모두 필터링 되므로 이러한 불용어 처리는 사용자에게 도움이 되기 보다는 사용자를 더 혼란스럽게 하는 기능일 수 있다.
    
    불용어 처리 자체를 완전히 무시하거나 MySQL 서버에 내장된 불용어 대신 사용자가 직접 불용어를 등록하는 방법을 권장한다.
    
    - 전문 검색 인덱스의 불용어 처리 무시 (두 가지 방법)
        1. 스토리지 엔진에 관계 없이 MySQL 서버의 모든 전문 검색 인덱스에 대해 불용어 완전 제거
            
            → MySQL 서버 설정 파일 `my.cnf` 의 `ft_stopword_file=''`
            
        2. InnoDB 스토리지 엔진을 사용하는 테이블의 전문 검색 인덱스에 대해서만 불용어 처리 무시
            
            → `innodb_ft_enable_stopword` 시스템 변수 OFF
            
    - 사용자 정의 불용어 사용 (두 가지 방법)
        1. 불용어 목록을 파일로 저장하고 파일 경로 등록
            
            → `ft_stopword_file='/data/custom_stopword.txt'`
            
        2. 불용어 목록 테이블로 저장 (InnoDB 스토리지 엔진 사용 테이블의 전문 검색 인덱스에서만 사용 가능)
            
            → 불용어 테이블 생성 후 `SET GLOBAL innodb_ft_sever_stopword_table='mydb/my_stopword'`
            
            또는 `innodb_ft_user_stopword_table` 시스템 변수 이용
            
    

### 8.5.2 전문 검색 인덱스의 가용성

전문검색 인덱스를 사용하려면 반드시 다음 두 조건을 갖춰야 한다.

- **쿼리 문장이 전문 검색을 위한 문법 (`MATCH … AGAINST …`)을 사용**
- **테이블이 전문검색 대상 칼럼에 대해서 전문 인덱스 보유**

## 8.6 함수 기반 인덱스

일반적인 인덱스는 칼럽의 값 일부 (칼럼 값의 앞부분) 또는 전체에 대해서만 인덱스 생성이 허용된다.

함수 기반 인덱스는 **칼럼의 값을 변형해서 만들어진 값에 대해 인덱스를 구축**해야 할 경우 사용한다. 인덱싱 값 계산 과정의 차이만 있고 인덱스의 내부 구조는 B-Tree 인덱스와 동일함.

MySQL 서버에서 함수 기반 인덱스를 구현하는 방법은 가상 칼럼을 이용하는 방법, 함수를 이용하는 방법 두 가지 이다.

### 8.6.1 가상 칼럼*을 이용한 인덱스

```sql
CREATE TABLE user(
	user_id BIGINT,
	first_name VARCHAR(10),
	last_name VARCHAR(10),
	PRIMARY KEY (user_id)
);
```

first_name + last_name으로 검색해야 하는 요건이 생겼다면 full_name이라는 칼럼을 추가하고 모든 레코드에 칼럼을 업데이트 하는 작업을 거쳐야 full_name 인덱스를 추가할 수 있었다.

MySQL 8.0버전부터는 다음과 같이 가상 칼럼을 추가하고 그 가상 칼럼에 인덱스를 생성할 수 있게 됐다.

```sql
ALTER TABLE user
	ADD full_name VARCHAR(30) AS (CONCAT(first_name,' ',last_name)) VIRTUAL,
	ADD INDEX ix_fullname (full_name);
```

가상 칼럼은 테이블에 새로운 칼럼을 추가하는 것과 같은 효과를 내기 때문에 실제 테이블의 구조가 변경된다는 단점이 있다.

가상 칼럼의 `VIRTUAL`과 `STROED` 옵션 차이는 15.8철 ‘가상 칼럼 (파생 칼럼)’ 참

.* **가상 칼럼 (Virtual Column)**: **실제 저장 공간을 차지하지 않으면서**, 다른 컬럼들의 값을 조합하거나 계산한 결과를 테이블 컬럼처럼 다룰 수 있다. 테이블의 메타데이터(구조 정보) 자체가 변경되므로 "테이블 데이터"는 늘리지 않지만 "테이블 구조"는 바꾼다.

### 8.6.2 함수를 이용한 인덱스

MySQL 8.0 버전부터는 **테이블 구조를 변경하지 않고 함수를 직접 사용**하는 인덱스를 생성할 수 있게 됐다.

```sql
CREATE TABLE user(
	user_id BIGINT,
	first_name VARCHAR(10),
	last_name VARCHAR(10),
	PRIMARY KEY (user_id),
	INDEX ix_fullname ((CONCAT(first_name,' ',last_name)))
)
```

함수 기반 인덱스를 제대로 활용하려면 **반드시 조건절에 함수 기반 인덱스에 명시된 표현식이 그대로 사용돼야 한다.** 함수 생성 시 명시된 표현식과 쿼리의 WHERE 조건절에 사용된 표현식이 다르다면 MySQL 옵티마이저는 다른 표현식으로 간주해서 함수 기반 인덱스를 사용하지 못한다.

```sql
가상 칼럼을 이용한 방법과 직접 함수를 이용한 인덱스는 내부적으로 동일한 구현 방법을 사용한다. 따라서 어떤 방법을 사용하더라도 둘의 성능 차이는 발생하지 않는다.
```

## 8.7 멀티 밸류 인덱스

전문 검색 인덱스를 제외한 모든 인덱스는 레코드 1건이 1개의 인덱스 키 값을 가진다.

하지만 멀티 밸류 (Multi-Value) 인덱스는 **하나의 데이터 레코드가 여러 개의 키 값을 가질 수 있는 형태**의 인덱스이다.

이는 일반적인 RDBMS를 기준으로 생각하면 정규화에 위배되지만, 최근 RDBMS들이 JSON 데이터 타입을 지원하면서 **JSON 배열 타입 필드**에 저장된 원소들에 대한 인덱스가 필요해졌다.

MySQL 8.0부터 MySQL 서버의 JSON 관리 기능은 태생적으로 JSON을 사용했던 MongoDB에 비해서도 부족함이 없는 상태가 되었다.

멀티 밸류 인덱스를 활용하기 위해서는 일반적인 조건 방식을 사용하면 안 되고, **반드시 다음 함수들을 이용해서 검색해야 옵티마이저가 인덱스를 활용한 실행 계획을 수립**한다.

- MEMBER OF()
- JSON_CONTAINS()
- JSON_OVERLAPS()

## 8.8 클러스터링 인덱스

MySQL 서버에서 클러스터링은 테이블의 레코드를 PK를 기준으로 비슷한 레코드끼리 묶어서 저장한다. 이는 주로 비슷한 값들을 동시에 조회하는 경우가 많다는 점에서 착안한 것이다.

MySQL에서 클러스터링 인덱스는 **InnoDB 스토리지 엔진에서만 지원**하며, 나머지 스토리지 엔진에서는 지원되지 않는다.

### 8.8.1 클러스터링 인덱스

테이블의 **PK에 대해서만 적용**되는 내용이다. PK 값이 비슷한 레코드끼리 묶어서 저장하는 것을 클러스터링 인덱스라고 표현한다. 인덱스 알고리즘이라기 보다는 테이블 레코드의 저장방식이다.

중요한 것은 **PK 값에 의해 레코드의 물리적 저장 위치가 결정**되고, **PK 값이 변경되면 그 레코드의 물리적 저장 위치 또한 변경**되어야 한다.

일반적으로 InnoDB와 같이 항상 클러스터링 인덱스로 저장되는 테이블은 PK 기반의 검색이 매우 빠르며, 대신 레코드의 저장이나 PK의 변경이 상대적으로 느리다.

![image (1)](https://github.com/user-attachments/assets/8c2aefa4-0433-41c0-8094-eea8a036043c)


브랜치 노드는 자식 페이지의 ‘최솟값(가장 첫 번째 인덱스 키)’을 기준으로, 자식 페이지를 가리키는 포인터를 저장함

PK가 없는 경우에는 InnoDB 스토리지 엔진이 **다음 우선순위대로 PK를 대체할 칼럼을 선택**한다.

1. PK가 있으면 PK를 클러스터링 키로 선택
2. NOT NULL 옵션의 UNIQUE INDEX 중 첫 번째 인덱스를 클러스터링 키로 선택
3. 자동으로 유니크한 값을 가지도록 증가되는 칼럼을 내부적으로 추가한 후 클러스터링 키로 선택
    
    → 이 경우 자동으로 추가된 PK 일련번호 칼럼은 사용자에데 노출되지 않으며, 쿼리 문장에서도 명시적으로 사용할 수 없다. 아무 의미 없는 숫자로 클러스터링 되는 것이기 때문에 우리에게 아무런 이점이 없다.
    

### 8.8.2 세컨더리 인덱스에 미치는 영향

MyISAM이나 MEMORY 테이블 같은 클러스터링 되지 않은 테이블은 PK나 세컨더리 인덱스는 구조적으로 아무런 차이가 없다.

InnoDB 테이블에서 세컨더리 인덱스가 실제 레코드가 저장된 주소를 가지고 있다면 클러스터링 키 값이 변경될 때마다 데이터 레코드의 주소가 변경ㄷ되고 그때마다 해당 테이블의 모든 인덱스에 저장된 주솟값을 변경해야 한다.

위와 같은 오버헤드를 제거하기 위해 InnoDB 테이블의 모든 세컨더리 인덱스는 해당 레코드가 저장된 주소가 아니라 PK 값을 저장하도록 구현돼 있다.

### 8.8.3 클러스터링 인덱스의 장점과 단점

MyISAM과 같은 클러스터링 되지 않은 일반 PK와 클러스터링 인덱스를 비교했을 때의 상대적인 장단점

- 장점
    - PK (클러스터링 키)로 검색할 때 처리 성능이 매우 빠름 (특히, PK 범위를 검색하는 경우)
    - 테이블의 모든 세컨더리 인덱스가 PK를 가지고 있기 때문에 인덱스만으로 처리될 수 있는 경우가 많음 (커버링 인덱스 → 10장에서 자세히)
- 단점
    - 테이블의 모든 세컨더리 인덱스가 클러스터링 키를 갖기 때문에 클러스터링 키 값의 크기가 클 경우 전체적으로 인덱스의 크기가 커짐
    - 세컨더리 인덱스를 통해 검색할 때 PK로 다시 한번 검색해야 하므로 처리 성능이 느림
    - INSERT 할 때 PK에 의해 레코드의 저장 위치가 결정되기 때문에 처리 성능이 느림
    - PK를 변경할 때 레코드를 DELETE하고 INSERT하는 작업이 필요하기 대문에 처리 성능이 느림
    

### 8.8.4 클러스터링 테이블 사용 시 주의사항

- **클러스터링 인덱스 키의 크기**
    
    클러스터링 테이블의 경우 모든 세컨더리 인덱스가 PK 값을 포함하므로 PK의 크기가 커지만 세컨더리 인덱스도 자동으로 크기가 커진다.
    
    일반적으로 테이블에 세컨더리 인덱스가 4~5개정도 생성된다는 것을 고려하면 세컨더리 인덱스 크기는 급격히 증가한다.
    

- **프라이머리 키는 AUTO-INCREMENT보다는 업무적인 칼럼으로 생성 (가능한 경우)**
    
    PK는 그 의미만큼이나 중요한 역할을 하기 때문에 대부분 검색에서 상당히 빈번하게 사용되는 것이 일반적이다. 그러므로 설령 그 칼럼의 크기가 크더라도 업무적으로 해당 레코드를 대표할 수 있다면 그 칼럼을 PK로 설정하는 것이 좋다.
    

- **프라이머리 키는 반드시 명시할 것**
    
    AUTO_INCREMENT 칼럼을 이용해서라도 PK는 생성하는 것을 권장한다. InnoDB 테이블에서 PK를 정의하지 않으면 InnoDB 스토리지 엔진이 내부적으로 일련번호 칼럼을 추가한다. 이 칼럼은 사용자가 전혀 접근할 수 없다. 결국 AUTO_INCREMENT 칼럼과 똑같지만 사용할 수 없는 값인 것이다.
    
- **AUTO-INCREMENT 칼럼을 인조 식별자로 사용할 경우**
    
    여러 개의 칼럼이 복합으로 PK가 만들어지는 경우 PK의 크기가 길어지더라도 세컨더리 인덱스가 필요치 않다면 그대로 PK를 사용하는 것이 좋다. 세컨더리 인덱스도 필요하고 PK 길이도 길다면 AUTO_INCREMENT 칼럼을 PK로 설정하여 인조식별자를 사용하면 된다. 로그 테이블 같이 조회보다는 INSERT 위주의 테이블들은 AUTO_INCREMENT를 이용한 인조 식별자를 PK로 설정하는 것이 성능 향상에 도움이 된다.
    

## 8.9 유니크 인덱스

유니크는 테이블이나 인덱스에 같은 값이 2개 이상 저장될 수 없는 제약조건이다. 유니크에 NULL도 저장될 수 있는데, NULL은 특정 값이 아니므로 2개 이상 저장될 수 있다.

**MySQL에서는 인덱스 없이 유니크 제약만 설정할 방법이 없다.** PK는 기본적으로 NOT NULL + UNIQUE이다.

### 8.9.1 유니크 인덱스와 일반 세컨더리 인덱스의 비교

유니크 인덱스와 유니크하지 않은 일반 세컨더리 인덱스는 인덱스 구조상 아무런 차이점이 없다.

- **인덱스 읽기**
    
    유니크하지 않은 세컨더리 인덱스에서 한 번 더 해야 하는 작업은 디스크 읽기가 아니라 CPU에서 칼럼값을 비교하는 작업이기 때문에 이는 성능상 영향이 거의 없다.
    
    즉, 유니크하지 않은 세컨더리 인덱스는 중복된 값이 허용되므로 읽어야 할 레코드가 많아서 느린 것이지, 인덱스 자체의 구조 때문에 느린 것이 아니다.
    
    하나의 값을 검색하는 경우 유니크 인덱스와 일반 세컨더리 인덱스는 사용된느 실행 계획이 다르다. 하지만 이는 **1개의 레코드를 읽는지, 2개 이상의 레코드를 읽는지의 차이만 있을 뿐 읽어야할 레코드 건수가 같아면 성능상 차이는 미미하다.**
    
- **인덱스 쓰기**
    
    새로운 레코드가 INSERT 또는 인덱스 칼럼 값이 변경되는 경우 인덱스 쓰기 작업이 필요하다.
    
    유니크 인덱스의 키 값을 쓸 때는 **중복된 값이 있는지 체크하는 과정이 추가로** **필요**하다. 그래서 유니크하지 않은 세컨더리 인덱스의 쓰기보다 느리다.
    
    MySQL에서는 유니크 인덱스에서 중복도니 값을 체크할 때는 읽기 잠금을 사용하고, 쓰기를 할 때는 쓰기 잠금을 사용하는데 이 과정에서 **데드락**이 빈번히 발생한다.
    
    또한, InnoDB 스토리지 엔진은 인덱스 키 저장 시 체인지버처를 통해 버퍼링하여 처리한다. 하지만 **유니크 인덱스는 반드시 중복 체크를 해야 하므로 작업 자체를 버퍼링하지 못한다.**
    

### 8.9.2 유니크 인덱스 사용 시 주의사항

유일성이 꼭 보장되어야 하는 칼럼이라면 유니크 인덱스를 생성해야하지만 성능 향상을 기대하고 불필요하게 유니크 인덱스를 생성하지 않는 것이 좋다.

유니크 인덱스는 쿼리 실행 계획이나 테이블 파티션에도 영향을 미친다 → 10장, 13장에서 자세히

## 8.10 외래키

MySQL에서 외래키는 InnoDB 스토리지 엔진에서만 생성할 수 있으며, FK 제약이 설정되면 자동으로 연관되는 테이블의 칼럼에 인덱스까지 생성된다. FK가 제거되지 않은 상태에서는 자동으로 생성된 인덱스를 삭제할 수 없다.

InnoDB의 외래키 관리에는 중요한 두 가지 특징이 있다.

- 테이블의 변경 (쓰기 잠금)이 발생하는 경우에만 잠금 대기가 발생한다.
- 외래키와 연관되지 않은 칼럼의 변경은 최대한 잠금 대기를 발생시키지 않는다.

### 8.10.1 자식 테이블의 변경이 대기하는 경우

자식 테이블에서는 부모 키를 참조하는 데이터를 추가하거나(FK 컬럼 INSERT), FK 컬럼 값을 수정할 때, 부모 테이블에 해당 키가 존재하는지 확인하는 과정에서 잠금 대기가 발생할 수 있다.

### 8.10.2 부모 테이블의 변경 작업이 대기하는 경우

부모 테이블에서는 키 값을 수정하거나 삭제할 때, 이를 참조하는 자식 테이블의 데이터를 확인하거나 동기화하는 과정에서 잠금 대기가 발생할 수 있다. 특히 부모 키를 수정하는 경우에는 자식 테이블 전체에 영향을 미칠 수 있기 때문에 주의가 필요하다.
