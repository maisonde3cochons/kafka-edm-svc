#### Kafka EDM SVC 

----------------------------

#### [주의사항!!! docker-compose 파일을 WLS2에서 실행 시 주요 이슈가 발생 부분]

##### 1. docker-compose 실행 path가 windows directory일 경우 volume mount(mysql/kafka/zookeeper) 문제가 발생한다
> 반드시 avoid!! <b>/mnt/c/ </b> 나 <b>/mnt/d/ </b> 에서 실행하지 않는다

##### 2. docker-compose 실행 후 생성되는 kafka/zooker의 권한을 꼭 확인한다
> docker container와 mount되는 path directory가 root권한으로 생성 되는 경우가 많다.
> 꼭 사전에 kafka/logs, zookeeper/data 폴더 생성 후 실행 user 권한으로 변경해준다
```
mkdir -p kafka/logs 
mkdir -p zookeeper/data
chown -R user:user zookeeper kafka
```

##### 3. /etc/hosts를 확인해서 사용할 도메인이 등록되어 있는지 확인한다
```
127.0.0.1 kafka zookeeper
```

##### 4. docker-compose 실행 후 docker container에 mount된 volume 권한을 변경하지 않는다
> docker-compose 재실행 시 권한 문제가 발생한다

![image](https://user-images.githubusercontent.com/30817824/170620369-16000fab-b9e1-47af-b95b-93e1cebf4282.png)


-----------------------


#### [설치]

##### 1. pip3 및 dependency 설치
```
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py
pip3 --version
pip3 install -r requirements.txt

#pip3 upgrade & confluent-kafka 설치
pip3 install --upgrade pip
pip3 install confluent-kafka

#mysql client 설치 (mysqlclient error: metadata-generation-failed 해결)
sudo apt-get install python3-dev default-libmysqlclient-dev build-essential
pip3 install mysqlclient==2.0.3

```

##### 2. node js 설치 및 버전 확인
```
apt-get install npm
npm --version
node --version
npm install mysql express
```
##### 3. /etc/hosts 설정(kafka1, zookeeper1추가)
```
127.0.0.1       localhost   kafka1  zookeeper1
```

----------------------------

#### [설정]

##### 1. create_topics.py 파일 확인
> ##### create_topics.py 파일은 topics를 만들어주는 python script임
```
#!/usr/bin/env python3
from confluent_kafka.admin import AdminClient, NewTopic

conf = {'bootstrap.servers': 'kafka1:19092'}
admin = AdminClient(conf=conf)
admin.create_topics([NewTopic('order', 1, 1), NewTopic('inventory', 1, 1), NewTopic('payment', 1, 1)])
print(admin.list_topics().topics)
```

##### 2. DB Table/Data 생성
```
# 인벤토리 테이블 생성
CREATE TABLE inventory (
    id int(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name char(30) NOT NULL,
    price int(10) NOT NULL,
    quantity int(10) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE inventory_history (
    id int(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    transaction_id char(36) NOT NULL,
    inventory_id int(10) NOT NULL,
    quantity int(10) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

# 아이템 입력
INSERT INTO inventory(name, price, quantity) VALUES('cloth', 50000, 100);
INSERT INTO inventory(name, price, quantity) VALUES('cap', 30000, 100);
INSERT INTO inventory(name, price, quantity) VALUES('sunglasses', 25000, 100);
INSERT INTO inventory(name, price, quantity) VALUES('necklace', 150000, 100);
INSERT INTO inventory(name, price, quantity) VALUES('earring', 15000, 100);
```

##### 3. order_service(8080 port) deploy 및 확인
> ##### deploy order service
```
# 주문 서비스 up
./order_service.py 
```

> ##### 확인
```
# 1건의 주문 요청
curl -v -XPOST http://localhost:8080/v1/order -H'Content-Type: application/json' \
--data-binary @- << EOF
{
    "order": {
        "user_id": "user",
        "name": "Hong Gil Dong",
        "addr": "29, Hansil-ro, Dalseo-gu, Daegu, Republic of Korea",
        "tel": "01012341234",
        "email": "hong@gil.com"
    },
    "inventory": {
        "id": 2, 
        "quantity": 5
    }
} 
EOF
```

##### 4. inventory_service 확인
> ##### deploy inventory consumer

```
# 인벤토리 서비스 컨슈머 up
./inventory_consumer.py
```

> ##### output (이전에 입력한 값이 출력됨)
```
b'{"order": {"user_id": "user", "name": "Hong Gil Dong", "addr": "29, Hansil-ro, Dalseo-gu, Daegu, Republic of Korea", "tel": "01012341234", "email": "hong@gil.com"}, "inventory": {"id": 2, "quantity": 5}, "transaction_id": "dcd355b4-dcd9-11ec-9ab4-00155d1575e4", "status": "ORDER_CREATED"}'
process_inventory_reserved
total_price: 150000
```

##### 5. payment_consumer 확인
> ##### deploy payment consumer

```
# 결제 서비스 컨슈머 up
./payment_consumer.py True
```

> ##### output (이전에 입력한 값이 출력됨)
```
b'{"order": {"user_id": "user", "name": "Hong Gil Dong", "addr": "29, Hansil-ro, Dalseo-gu, Daegu, Republic of Korea", "tel": "01012341234", "email": "hong@gil.com"}, "inventory": {"id": 2, "quantity": 5}, "transaction_id": "dcd355b4-dcd9-11ec-9ab4-00155d1575e4", "status": "INVENTORY_RESERVED", "payment": {"total_price": 150000}}'
```

##### 6. payment_consumer 확인

> ##### deploy order service
```
# 주문 서비스 up
./order_service.py 
```

> ##### 주문현황 조회(transaction_id 및 user 입력)

```
# 주문현황 조회
curl -v -XGET http://localhost:8080/v1/order?transaction_id=${TRANSACTION_ID}&user_id=user
curl -v -XGET http://localhost:8080/v1/order?transaction_id=dcd355b4-dcd9-11ec-9ab4-00155d1575e4&user_id=user
curl -v -XGET http://localhost:8080/v1/order?transaction_id=1ea26836-dcdf-11ec-9ab4-00155d1575e4user_id=user2
```

