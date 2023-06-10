---
title: "MongoDB 인덱스"
description: MongoDB 인덱스는 왜 필요한지, 어떤 방식으로 동작하는지 알아봅시다.
slug: mongodb-indexes
date: 2023-06-05T15:11:17+09:00
image: /img/mongodb.png
categories:
    - MongoDB
draft: false
---

## Random I/O vs Sequential I/O
Random I/O
* 디스크에 무작위로 위치한 데이터를 읽고 쓰는 것을 말합니다.
* 디스크 헤드가 물리적으로 디스크의 여러 위치로 이동해야 하기 때문에 일반적으로 느립니다.

Sequential I/O
* 디스크의 특정 영역에 연속적으로 위치한 데이터를 읽고 쓰는 것을 말합니다.
* 디스크 헤드가 물리적으로 연속된 위치로만 이동하면 되므로 일반적으로 빠르고 효율적입니다.

우리는 Random I/O의 횟수를 줄이는 방향으로, 즉 꼭 필요한 데이터만 읽도록 쿼리를 개선해야 합니다.

## Index
인덱스가 없는 상태에서는 특정 쿼리를 수행하기 위해 전체 데이터를 훑어야 하므로 이는 크게 Random I/O를 초래합니다.
디스크 헤드는 모든 데이터 블록을 무작위로 방문해야 하기 때문입니다.  
반면에, 인덱스가 존재하는 경우, 인덱스를 사용해 필요한 데이터를 빠르게 찾을 수 있습니다. 
디스크의 연속된 영역에 위치한 인덱스 블록을 순차적으로 읽음으로써 Sequential I/O가 가능하게 합니다.  
위에서 말한 Random I/O를 줄이는 방법으로 사용할 수 있다는 것입니다.  
(물론 Non-Clustered Index에서 주소가 가르키는 데이터를 가져오는 작업은 random I/O로 분류됩니다.)

### Clustered Index
```sh
db.createCollection("collection",{
    clusteredIndex: {
        "key": { _id: 1 },
        "unique": true,
        "name": "clustered key"
    }
});
```
MongoDB 5.3부터는 [clustered index](https://www.mongodb.com/docs/manual/reference/method/db.createCollection/#std-label-db.createCollection.clusteredIndex)를 가진 컬렉션을 생성할 수 있게 되었습니다.
* 일반 인덱스와 가장 다른 차이점은 인덱스 키의 순서대로 인덱스와 데이터가 물리적으로 저장된다는 것입니다.  
  (일반 인덱스는 참조를 사용해서 데이터 위치를 가르킵니다.)  
그러므로 clustered index는 별도의 인덱스 저장 공간이 필요 없습니다.  
insert, update, delete 시 _id 인덱스에 별도 쓰기가 필요 없으므로 한번의 쓰기만을 필요로 합니다.
* 특히 range query에서 유용합니다.
인덱스 키의 순서에 따라 물리적으로 저장된 데이터를 순차적으로 읽을 수 있게 됩니다.
* insert 시 데이터가 순서대로 저장을 해야하기 때문에 추가적인 I/O 작업이 있을 수 있습니다.
문서에서는 _id에 sequantial한 키를 포함하는 것을 권장합니다.

### None-Clustered Index
![non-clustered index](mongodb-indexes1.svg)
Primary index, Secondary index와 같이 clustered index가 아닌 인덱스를 말합니다.
##### 동작 방식
* *[WiredTiger 스토리지 엔진](https://www.mongodb.com/docs/manual/core/wiredtiger/)에서는 기본적으로 B-Tree 기반의 인덱싱 알고리즘을 사용하고 있습니다.  
B-Tree 인덱스는 데이터를 물리적으로 정렬된 방식으로 저장하지는 않지만, 정렬된 키와 그 키에 연관된 데이터의 물리적 주소를 가지고 있습니다.
* 인덱스 키를 추가할 때는 B-Tree의 리프노드에 값을 추가하고 데이터가 저장된 위치를 저장합니다.  
테이블에 레코드를 추가하는 작업의 비용이 1, 1개의 인덱스가 있다고 가정하면 2(1*1 + 1)정도의 비용이 든다고 대략적으로 예측할 수 있습니다.
* 인덱스 키를 삭제할 때는 해당 키 값을 찾아서 삭제 마크를 합니다. 즉, 물리적으로 제거되는 것이 아니라, 삭제 마크를 사용하여 논리적으로 삭제된 것으로 표시합니다.  
마킹된 항목은 실제로 물리적으로 제거되거나 재사용될 수 있습니다.  
인덱스 키를 삭제하면 WiredTiger 스토리지 엔진은 Write-Ahead Log에 기록합니다. 실제 디스크에 이 변경 사항이 적용되는 시점은 `syncPeriodSecs`라는 설정으로 제어할 수 있습니다.
* 인덱스 키를 수정할 때는 먼저 키 값을 삭제하고 새로운 키 값을 추가하는 방식으로 처리됩니다.
* 인덱스 키를 검색할 때는 B-Tree의 루트 노드부터 최종 리프 노드까지 비교하는 과정을 통해서 이동합니다.  
B-Tree 인덱스를 이용한 검색은 100% 일치, 앞 부분 일치, 그리고 range query 등에 사용될 수 있습니다.
* 인덱스 키 값의 사이즈는 작을수록 좋습니다.  
키 값의 사이즈가 커지면 하나의 인덱스 페이지가 담을 수 있는 인덱스 키 값의 개수가 작아지고, 이에 따라 간접적으로 B-Tree의 깊이가 더 깊어질 수 있습니다. 이는 디스크 I/O가 늘어나게 되어 성능 저하를 가져올 수 있습니다.

*WiredTiger 스토리지 엔진은 MongoDB 3.2부터 기본 엔진입니다.

##### 메모리
* 빠른 성능을 위해 [인덱스가 RAM 용량 내에 들어가게 하는 전략](https://www.mongodb.com/docs/manual/tutorial/ensure-indexes-fit-ram/)을 취할 수 있습니다. (디스크에서 인덱스를 읽지 않도록)
* RAM에 들어갈 수 있는 인덱스의 양이 제한적이기 때문에, 인덱스의 크기가 RAM의 크기를 초과하면 MongoDB는 인덱스 데이터를 디스크에서 가져와야 합니다.
* 인덱싱된 필드의 값이 삽일될 때마다 증가하고 대부분의 쿼리가 최근에 추가된 문서를 선택하는 경우 가장 최근을 포함하는 인덱스 부분만 RAM에 유지하면 됩니다.

### TTL Index
```sh
db.collection.createIndex(
  { "createdDate": 1 }, 
  { expireAfterSeconds: 3600} 
);
```
특정 시간이 지나면 컬렉션에서 문서를 자동으로 제거하는 [Index](#none-clustered-index)입니다.
* TTL 인덱스로 데이터를 삭제하는 작업은 주로 오래된 데이터(메모리에 캐시되지 않은 데이터)가 될 가능성이 높은데, 이 경우 디스크를 읽어서 처리해야하는 경우가 많습니다.
* TTL Monitor라는 별도의 백그라운드에서 도는 thread에서 동작합니다.
* 용량이 크거나 많은 인덱스를 가진 컬렉션에서 많은 도큐먼트가 삭제된다면 많은 디스크의 I/O를 유발하고 복제 지연을 발생시킬 수 있습니다. (도큐먼트와 연결된 인덱스 키도 모두 제거되어져야하기 때문입니다.)

## Conclusion
인덱스를 사용하면 쿼리의 성능 등을 향상시킬 수 있습니다.  
그러나 인덱스를 저장하기 위해 상당한 디스크 공간을 사용할 수 있고, insert, update의 경우 인덱스를 위한 추가 작업으로 부하를 더할 수 있습니다.  
application의 쿼리 패턴과 데이터 모델, 사용하는 서버 스펙(메모리, 디스크)을 고려하여 인덱스를 사용하는 것이 좋습니다.

## 참조
* [MongoDB > Indexes](https://www.mongodb.com/docs/manual/indexes/)
* [Real MongoDB](https://www.yes24.com/Product/Goods/58142119)