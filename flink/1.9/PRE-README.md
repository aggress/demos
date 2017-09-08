Open at least 4 different terminal windows to make life eaiser


Confluent Kafka Packages
-----------------------

In all cases we're ensuring the service name matches what the demo is expecting, in this case `kafka`
rather than `confluent-kafka`

1. `$ dcos package install beta-confluent-kafka --name="kafka" --yes`
2. Create the Schema Registry JSON definition
```
$ tee schema-registry.json << EOF
{
  "registry": {
    "kafkastore": "dcos-service-kafka"
  }
}
EOF
```
3. `$ dcos package install confluent-schema-registry --options=schema-registry.json --yes`
4. Create the REST Proxy JSON definition
```
$ tee rest-proxy.json << EOF
{
  "proxy": {
    "kafka-service": "kafka",
    "zookeeper-connect": "master.mesos:2181/dcos-service-kafka",
    "schema-registry-service": "schema-registry.marathon.l4lb.thisdcos.directory:8081"
  }
}
EOF
```
5. `$ dcos package install confluent-rest-proxy --options=rest-proxy.json`
6. `$ dcos beta-confluent-kafka --name="kafka" topic create fraud`
7. `$ dcos beta-confluent-kafka --name="kafka" topic create transactions`


Mongo
---------------

1. `$ dcos package repo add --index=0 mongo-aws https://s3-eu-west-1.amazonaws.com/dcosmongodb/packages/stub-universe-mongo.json`
1. `$ dcos package install mongo`
1. Find one of the endpoints (default to port 27017) like 10.0.0.127:27017
1. `$ dcos node ssh --master-proxy --leader`
1. `$ sudo su -`
1. `$ curl https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-3.4.7.tgz | tar xvz`
1. `$ export PATH=$PATH:/root/mongodb-linux-x86_64-3.4.7/bin`
1. `$ mongo useradmin:useradminpassword@10.0.0.127:27017/admin`
1. Whilst in the mongo shell run the following
```
use demo
db.createUser(
   {
     user: "test",
     pwd: "test",
     roles: [ "readWrite", "dbAdmin" ]
   }
)
```
10. `exit`
1. Test connecting with the new user `mongo test:test@10.0.0.127:27017/demo`
1. Add a sample collection and document, then read it out
```
db.fraud.insert( { item: "foo", qty: 1 } )
db.fraud.find()
db.transactions.insert( { item: "bar", qty: 1 } )
db.transacation.find()
```


Connect
----------------

The confluent-connect package ships with a handful of official connectors, whilst there are many
other community connectors available. These are external JARs which need importing and enabling.
Getting those into the official package is for another day, here's one that works.

We'll run this from the leading master again

1. `dcos node ssh --master-proxy --leader`
1. Make a note of the private IP of the master you connect to and amend the following before launching the container
```
$ docker run -d --net=host \
       -e ID=01 \
       -e BS=broker.kafka.l4lb.thisdcos.directory:9092 \
       -e ZK=master.mesos:2181/dcos-service-kafka \
       -e SR=http://schema-registry.marathon.l4lb.thisdcos.directory:8081 \
       -e HOST=10.0.4.148 \
       landoop/fast-data-dev-connect-cluster
```
3. Create the Mongo sink connector definition, amend `connect.mongo.connection` for the correct IP
```
$ tee mongo_sink.json << EOF
{
    "name": "MongoSinkConnector",
    "config": {
        "connector.class": "com.datamountaineer.streamreactor.connect.mongodb.sink.MongoSinkConnector",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "false",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "key.converter.schemas.enable": "false",
        "topics": "fraud",
        "tasks.max": "1",
        "connect.mongo.connection": "mongodb://10.0.0.127:27017",
        "connect.mongo.db": "demo",
        "connect.mongo.kcql": "INSERT INTO fraud SELECT * FROM fraud",
        "connect.mongo.username": "test",
        "connect.mongo.password": "test"
 }
}
EOF
```
4. Connect exposes a REST interface on :8083 (not the same as the Confluent REST Proxy) to allow you to interract with it.
We need to POST the new Mongo connector definition, but before you do that, switch to a new terminal window to tail the Connect
log for debugging purposes:
```
$ docker exec -it $(docker ps | grep landoop | awk {'print $1'}) /bin/bash
tail -f /var/log/connect-distributed.log
```
5. Switch back to the shell on the master and add the connector definition. Change the target IP to your master where Connect is running
```
$ curl -X POST -H "Content-Type: application/json" -H "Accept: application/json" --data @mongo_sink.json http://10.0.4.148:8083/connectors
```
You'll either see JSON output confirming it was valid, or a 500 if not.  In both cases see the log to check for any syntax errors and to
confirm that the connector has connected to Mongo correctly, look for words like success and connected rather than failure.

If FUBAR, then DELETE the connector, correct the issue and POST again
```
$ curl http://10.0.4.148:8083/connectors
$ curl -X DELETE curl http://10.0.4.148:8083/connectors/MongoSinkConnector
```

Testing
------------

At this point the Connect Mongo sink connector is live and connected to Mongo.  Now we want to produce a couple
of messages manually into Kafka, then check Mongo to see if they made it through.

1. Confirm the rest proxy endpoint, it should be `rest-proxy.marathon.l4lb.thisdcos.directory:8082`
1. From your master, POST a test message into the fraud topic
```
$ curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" \
      -H "Accept: application/vnd.kafka.v2+json" \
      --data '{"records":[{"value":{"foo":"bar"}}]}' "http://rest-proxy.marathon.l4lb.thisdcos.directory:8082/topics/fraud"
```
Check the Connect log you're still tailing, you'll see an entry like
```
[2017-09-08 11:34:40,606] INFO Opened connection [connectionId{localValue:5, serverValue:5243}] to mongo-0-mongod.mongo.autoip.dcos.thisdcos.directory:27017 (org.mongodb.driver.connection:71)
```
Check Mongo to see if it appeared
```
rs:PRIMARY> db.fraud.find()
{ "_id" : ObjectId("59b27e13252af579b5735f92"), "item" : "foo", "qty" : 1 }
{ "_id" : ObjectId("59b280504a68ad001a44d1e4"), "foo" : "bar" }
```

Next
----------

Continue the Flink demo setup from 

https://github.com/dcos/demos/tree/master/flink/1.9#generator