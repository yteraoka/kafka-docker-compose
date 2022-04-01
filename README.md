# docker compose を使って Kafka Cluster を組む

https://github.com/bitnami/bitnami-docker-kafka をベースにカスタマイズ

- zookeeper 1 node
- kafka broker 4 node
- TLS は無効
- kafka の version は 2.8
  - https://github.com/bitnami/bitnami-docker-kafka/tree/master/2.8/debian-10
- kafka client としての container を一つ追加

## 設定

bitnami の repository にあるものからの変更点

```
offsets.topic.replication.factor=3
transaction.state.log.min.isr=2
transaction.state.log.replication.factor=3
default.replication.factor=3
min.insync.replicas=2
acks=all
replica.lag.time.max.ms=5000
```

## 実行

```
docker compose up -d
```

掃除

```
docker compose down
docker volume rm kafka-docker-compose_kafka_{0,1,2,3}_data kafka-docker-compose_zookeeper_data
```

## テスト

```
docker exec -it kafka-docker-compose-client-1 bash
```

### Topic 作成

```
/opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server kafka-0:9092 \
  --create --topic test1 \
  --partitions 10 \
  --replication-factor 3
```

### consumer を起動

```
/opt/bitnami/kafka/bin/kafka-consumer-perf-test.sh \
  --bootstrap-server kafka-0:9092 \
  --topic test1 \
  --group group1 \
  --from-latest \
  --messages 1000 \
  --timeout 300000 \
  --print-metrics
```

### 別の Terminal で producer を起動

```
/opt/bitnami/kafka/bin/kafka-producer-perf-test.sh \
  --producer-props acks=all bootstrap.servers=kafka-0:9092 \
  --record-size 100 \
  --throughput 5 \
  --num-records 1000 \
  --topic test1 \
  --print-metrics
```

### Topic の状態確認

```
/opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server kafka-0:9092 --list
```

```
/opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server kafka-0:9092 --describe --topic test1
/opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server kafka-0:9092 --describe --topic __consumer_offsets
```

### consumer group の操作

#### consumer group の一覧取得

```
/opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server kafka-0:9092 \
  --list
```

#### consumer group の active な member を確認

```
/opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server kafka-0:9092 \
  --describe --members --group group1
```

出力例

```
GROUP           CONSUMER-ID                                            HOST            CLIENT-ID         #PARTITIONS
group1          consumer-group1-1-997d759c-3aef-4f0f-9d78-564dc94f7491 /172.23.0.3     consumer-group1-1 10
```

#### consumer group の 削除

```
/opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server kafka-0:9092 \
  --delete \
  --group group1
```

他に offset を変更したりすることもできる

## Sarama の tool でテスト

https://github.com/Shopify/sarama

- [examples](https://github.com/Shopify/sarama/tree/main/examples)
- [tools](https://github.com/Shopify/sarama/tree/main/tools)

### build

```
git clone https://github.com/Shopify/sarama.git
for d in sarama/tools/kafka-*; do
  if [ -d "$d" ]; then
    name=$(basename $d)
    (cd $d && GOOS=linux GOARCH=amd64 go build && docker cp $name kafka-docker-compose-client-1:/tmp/)
  fi
done
(cd sarama/examples/consumergroup && GOOS=linux GOARCH=amd64 go build && docker cp consumer kafka-docker-compose-client-1:/tmp/)
```

### consumer

ConsumerGroup を使う

```
/tmp/consumer -brokers kafka-0:9092 -topics test1 -group group1
```

### kafka-console-producer

```
/tmp/kafka-console-producer -brokers kafka-0:9092 -topic test1 -value test123 [-silent]
```

- `-value` で指定したメッセージを送信する
- `-partition` で partition を指定可能
- `-partitioner` で `manual` か `random` を指定可能
- `-key` や `-headers` も指定可能

### kafka-console-consumer

```
/tmp/kafka-console-consumer -brokers kafka-0:9092 -topic test1 [-verbose]
```

- `-partitions` で特定の partition だけを対象にできる (default は `all`)
- `-offset` は `newest` か `oldest` を指定可能 (default は `newest`)

### kafka-console-partitionconsumer

```
/tmp/kafka-console-partitionconsumer -brokers kafka-0:9092 -topic test1 -partition 0
```

- `-partition` 指定が必須