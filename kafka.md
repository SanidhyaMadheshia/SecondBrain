
 Kafka 4.x no longer needs ZooKeeper, so a single container is enough:
  
  docker run -d --name kafka -p 9092:9092 apache/kafka:latest
  
  Create a topic with 3 partitions and look at it:
  
  docker exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 \
    --create --topic orders --partitions 3 --replication-factor 1
  
  docker exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe --topic orders
  
  Play with the CLI tools before writing Go. Open two terminals:
  
  # terminal 1: consumer
  docker exec -it kafka /opt/kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 --topic orders --from-beginning \
    --property print.key=true --property print.partition=true
  
  # terminal 2: producer (type  key:value  and press enter)
  docker exec -it kafka /opt/kafka/bin/kafka-console-producer.sh \
    --bootstrap-server localhost:9092 --topic orders \
    --property parse.key=true --property key.separator=:
  
  Observe: send u1:a, u1:b, u2:c. Both u1 records land in the same partition.
