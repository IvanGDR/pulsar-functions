# pulsar-functions

Pulsar Functions are lightweight computational processes that consume messages from one or more topics, apply a user-supplied function, and then publish the result to one or more topics.

Users can develop simple self-contained pieces of code and then deploy them, and Pulsar takes care of the underlying details required to run and scale your code with high availability, including the Pulsar consumer and producers for these input and output topics 

You can write in either Java or the python

Pulsar provides simple computational benefits on the messages before routing it to the consumers. 

It is possible to have the output topic of one Pulsar Function be the input topic of another 

Stream processing
The term stream processing generally refers to the processing of unbounded datasets that are streamed in continuously from some source system. One of the key architectural decisions within a stream processing framework is how and when to process these endless datasets. There are three basic approaches to processing these datasets and can be implemented in functions are, batch processing, micro-batching, and streaming. 

Batch processing: It processes data in large groups, and it can be scheduled hourly or daily. This is used where real-time data processing is not required. 

Drawbacks: Processing latency

In cloud reducing the number of operations can lead to cost savings. Batching can help minimize the number of transactions, API calls, or database operations.

Ideal for cost saving applications.

Micro batching: It is a hybrid approach between batch and streaming, where it processes it in small batches at a very short interval. 

Reduces the latency compared to traditional batch processing. 

It is near real time processing

Streaming: Streaming involves processing data in real-time as it is ingested.

Use cases:
Data transformation (formatting incoming data).

Enrichment of messages (adding metadata or external information).

Real-time analytics and alerting (triggering based on specific conditions).

Streaming data pipelines (chaining functions for processing).

Stateless and Stateful Functions:
Stateless Functions: Functions that don’t need to keep track of any state or context between executions.

Stateful Functions: Functions that maintain state between invocations are useful for aggregations, counting, or any operation that needs memory of previous events.

Create a standalone pulsar cluster with pulsar manager in ctool: 
Create a docker network using the below command 



docker network create pulsar-network
Since we run the pulsar manager and the pulsar cluster it is important to run both the containers in single network

Create a standalone cluster 



docker run -d --name pulsar-standalone --network pulsar-network -p 6650:6650 -p 8080:8080 apachepulsar/pulsar:latest bin/pulsar standalone
Create pulsar manager 



docker run -it -d --network pulsar-network \
    -p 9527:9527 -p 7750:7750 \
    -e SPRING_CONFIGURATION_FILE=/pulsar-manager/pulsar-manager/application.properties \
    --link 033f086665e1 \
    apachepulsar/pulsar-manager:latest
Once you have both the containers Up and running 



docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                                                                                  NAMES
811f31cff20a   apachepulsar/pulsar-manager:latest   "/pulsar-manager/ent…"   33 minutes ago   Up 33 minutes   0.0.0.0:7750->7750/tcp, :::7750->7750/tcp, 0.0.0.0:9527->9527/tcp, :::9527->9527/tcp   recursing_gould
033f086665e1   apachepulsar/pulsar:latest           "bin/pulsar standalo…"   39 minutes ago   Up 22 minutes   0.0.0.0:6650->6650/tcp, :::6650->6650/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   pulsar-standalone
To enable functions-worker running as part of a broker, you need to set functionsWorkerEnabled to true in the /pulsar/conf/broker.conf file and restart the container.



docker restart pulsar-standalone
Run the below command in the pulsar-standalone docker container and get the serviceUrl and the brokerServiceUrl to configure/create and environment in the UI



pulsar/bin/pulsar-admin clusters get standalone
{
  "serviceUrl" : "http://localhost:8080",
  "brokerServiceUrl" : "pulsar://localhost:6650",
  "brokerClientTlsEnabled" : false,
  "tlsAllowInsecureConnection" : false,
  "brokerClientTlsEnabledWithKeyStore" : false,
  "brokerClientTlsTrustStoreType" : "JKS",
  "brokerClientTlsKeyStoreType" : "JKS"
}
The above shows localhost, since I am running this in ctool environment we need to expose the url to IPaddress of node inorder to connect from the local machine. Use the below command to update the IPaddress



/bin/pulsar-admin clusters update --url http://<IPADDRESS>:8080 --broker-url pulsar://<IPADDRESS>:6650 standalone
033f086665e1:/pulsar/conf$ /pulsar/bin/pulsar-admin clusters get standalone
{
  "serviceUrl" : "http://10.166.73.122:8080",
  "brokerServiceUrl" : "pulsar://10.166.73.122:6650",
  "brokerClientTlsEnabled" : false,
  "tlsAllowInsecureConnection" : false,
  "brokerClientTlsEnabledWithKeyStore" : false,
  "brokerClientTlsTrustStoreType" : "JKS",
  "brokerClientTlsKeyStoreType" : "JKS"
}
Create an ssh tunnel from your local to the ctool environment using below command 



ssh -i ssh.key -L 9527:localhost:9527 automaton@10.166.73.122
Access the pulsar admin console from using the link http://localhost:9527/#/environments

Auth is enabled by default and there is no user ID password to access the UI when you use latest version of the pulsar-manager. You can create a super-user using the following command. Then you can use the super user credentials to log in the Pulsar Manager UI.



CSRF_TOKEN=$(curl http://backend-service:7750/pulsar-manager/csrf-token)
curl \
    -H "X-XSRF-TOKEN: $CSRF_TOKEN" \
    -H "Cookie: XSRF-TOKEN=$CSRF_TOKEN;" \
    -H 'Content-Type: application/json' \
    -X PUT http://backend-service:7750/pulsar-manager/users/superuser \
    -d '{"name": "admin", "password": "apachepulsar", "description": "test", "email": "username@test.org"}'
Access the UI using the userid and password provided above and add environment using the serviceUrl and brokerUrl

Create Required Entities
 
You can create all these entities from the UI. Additionally, you can create the tenant, namespaces and topics using the below command.



#Create tenant
docker exec -it pulsar-standalone /pulsar/bin/pulsar-admin tenants create testing-tenant
#Create namespaces
docker exec -it pulsar-standalone /pulsar/bin/pulsar-admin namespaces create testing-tenant/tnamespace
#create input topics (you will have an option to create topics automatically as well)
docker exec -it pulsar-standalone /pulsar/bin/pulsar-admin topics create "persistent://testing-tenant/tnamespace/test-topic"
#create output topics
docker exec -it pulsar-standalone /pulsar/bin/pulsar-admin topics create "persistent://testing-tenant/tnamespace/function-topic"
Create a function in python



from pulsar import Function
import re
class ValidationProcessor(Function):
    def process(self, input, context):
        message = input
        context.get_logger().info(f"Received message: {message}")
        # Use regex to extract the number at the end of the message
        match = re.search(r'(\d+)$', message)
        if match:
            number = int(match.group(1))
            if number % 2 == 0:
                new_message = f"{message} EVEN"
            else:
                new_message = f"{message} ODD"
        else:
            new_message = f"{message} (no number found)"
        # Log the processed message
        context.get_logger().info(f"Processed message: {new_message}")
        # Publish the processed message to the output topic
        context.publish("persistent://public/default/function-topic", new_message.encode('utf-8'))
        return new_message
Copy the function to docker container. 

The above function just reads all the input that it gets and check the last number is even or odd and the writes its decision to a new topic called function topic.

Deploy the Function

Pulsar provides two modes to deploy a function:

cluster mode (for production) - you can submit a function to a Pulsar cluster and the cluster will take charge of running the function.

localrun mode - you can determine where a function runs, for example, on your local machine.

Deploy the function code in the cluster 



/pulsar/bin/pulsar-admin functions create \
  --tenant testing-tenant \
  --namespace tnamespace \
  --name validate \
  --py /pulsar/functions/test-function.py \
  --classname test-function.ValidationProcessor \
  --inputs persistent://testing-tenant/tnamespace/test-topic \
  --output persistent://testing-tenant/tnamespace/function-topic \
  --parallelism 1 ( optional to deploy the function on how many instances)
Troubleshooting a function:

Check the function status using the below command after you deployed and the example output should look like this 



/pulsar/bin/pulsar-admin functions status --tenant testing-tenant --namespace tnamespace --name validate
{
  "numInstances" : 1,
  "numRunning" : 1,
  "instances" : [ {
    "instanceId" : 0,
    "status" : {
      "running" : true,
      "error" : "",
      "numRestarts" : 0,
      "numReceived" : 0,
      "numSuccessfullyProcessed" : 0,
      "numUserExceptions" : 0,
      "latestUserExceptions" : [ ],
      "numSystemExceptions" : 0,
      "latestSystemExceptions" : [ ],
      "averageLatency" : 0.0,
      "lastInvocationTime" : 0,
      "workerId" : "c-standalone-fw-localhost-8080"
    }
  } ]
}
We should see the above running = true if it is not running we need to check the function logs where in the below locations /pulsar/logs/functions/testing-tenant/tnamespace/test-function and the log fine name will consist of the function name as shown test-function-0.log

The below get command will give information about the function deployed in detail 



/pulsar/bin/pulsar-admin functions get --tenant testing-tenant --namespace tnamespace --name test-function
{
  "tenant": "testing-tenant",
  "namespace": "tnamespace",
  "name": "test-function",
  "className": "test_function.ValidationProcessor",
  "inputSpecs": {
    "persistent://testing-tenant/tnamespace/test-topic": {
      "isRegexPattern": false,
      "schemaProperties": {},
      "consumerProperties": {},
      "poolMessages": false
    }
  },
  "output": "persistent://testing-tenant/tnamespace/function-topic",
  "producerConfig": {
    "useThreadLocalProducers": false,
    "batchBuilder": "",
    "compressionType": "LZ4"
  },
  "processingGuarantees": "ATLEAST_ONCE",
  "retainOrdering": false,
  "retainKeyOrdering": false,
  "forwardSourceMessageProperty": true,
  "userConfig": {},
  "runtime": "PYTHON",
  "autoAck": true,
  "parallelism": 1,
  "resources": {
    "cpu": 1.0,
    "ram": 1073741824,
    "disk": 10737418240
  },
  "cleanupSubscription": true,
  "subscriptionPosition": "Latest"
If you are running the pulsar in cluster mode and want to check on how many function workers we have in the cluster



bin/pulsar-admin functions-worker get-cluster
you can setup function workers in the broker node or if isolation is required we can setup the worker outside the broker.

You can update, stop, restart, reload the functions using the pulsar admin 

Produce and consume messages using pulsar admin

Produce some messages to Test topic



 /pulsar/bin/pulsar-client produce -m "Hello Pulsar 11" -n 10 persistent://testing-tenant/tnamespace/test-topic
Consume messages from output topic using a subscription



/pulsar/bin/pulsar-client consume persistent://testing-tenant/tnamespace/function-topic -s function-sub -n 0
Using simple python code

The below code will produce some random messages to the input-topic and consume it using a subscription called function-sub 



import pulsar
# Replace with the service URL of your Pulsar broker
service_url = 'pulsar://localhost:6650'
# Replace with your topic name where you want to produce your message
input_topic = 'persistent://testing-tenant/tnamespace/input-topic'
# Replace with your topic name you want to consume messages from
consumer_topic = 'persistent://testing-tenant/tnamespace/function-topic'
# Replace with your subscription name
subscription_name = 'function-sub'
def producer():
    # Create a Pulsar client
    client = pulsar.Client(service_url)
    # Create a producer
    producer = client.create_producer(input_topic)
    try:
        # Send a few messages
        for i in range(40,50):
            message = f'Hello Pulsar {i}'
            producer.send(message.encode('utf-8'))
            print(f'Sent message: {message}')
    finally:
        producer.close()
        client.close()
def consumer():
    # Create a Pulsar client
    client = pulsar.Client(service_url)
    # Create a consumer on the given topic and subscription
    consumer = client.subscribe(consumer_topic, subscription_name)
    try:
        while True:
            msg = consumer.receive()
            try:
                # Acknowledge and print the message
                print("Received message: '{}'".format(msg.data().decode('utf-8')))
                consumer.acknowledge(msg)
            except Exception as e:
                # If there is an error, don't acknowledge the message
                print("Failed to process message: '{}', error: {}".format(msg.data().decode('utf-8'), e))
                consumer.negative_acknowledge(msg)
    finally:
        consumer.close()
        client.close()
if __name__ == '__main__':
    producer()
    consumer()  
The above code will produce the messages Hello Pulsar <number> to our input topic (test-topic)

The function reads the topic and decides if the number is even or odd

The function then writes its decision to new topic function-topic

Then consumer subscribes to new topic using subscription name as function-sub

And the consumer prints the message as received as seen below



Sent message: Hello Pulsar 40
Sent message: Hello Pulsar 41
Sent message: Hello Pulsar 42
Sent message: Hello Pulsar 43
Sent message: Hello Pulsar 44
Sent message: Hello Pulsar 45
Sent message: Hello Pulsar 46
Sent message: Hello Pulsar 47
Sent message: Hello Pulsar 48
Sent message: Hello Pulsar 49
2024-07-25 13:25:13.741 INFO  [0x1f6ac8c00] ProducerImpl:800 | [persistent://testing-tenant/tnamespace/test-topic, standalone-32-29] Closing producer for topic persistent://testing-tenant/tnamespace/test-topic
2024-07-25 13:25:13.742 INFO  [0x16d493000] ProducerImpl:764 | [persistent://testing-tenant/tnamespace/test-topic, standalone-32-29] Closed producer 0
2024-07-25 13:25:13.742 INFO  [0x1f6ac8c00] ClientImpl:666 | Closing Pulsar client with 0 producers and 0 consumers
2024-07-25 13:25:13.742 INFO  [0x16d5ab000] ClientConnection:1320 | [[::1]:51065 -> [::1]:6650] Connection disconnected (refCnt: 1)
2024-07-25 13:25:13.742 INFO  [0x16d5ab000] ClientConnection:275 | [[::1]:51065 -> [::1]:6650] Destroyed connection to pulsar://localhost:6650-0
2024-07-25 13:25:13.742 INFO  [0x1f6ac8c00] ProducerImpl:757 | Producer - [persistent://testing-tenant/tnamespace/test-topic, standalone-32-29] , [batching  = off]
2024-07-25 13:25:13.743 INFO  [0x1f6ac8c00] Client:86 | Subscribing on Topic :persistent://testing-tenant/tnamespace/function-topic
2024-07-25 13:25:13.743 INFO  [0x1f6ac8c00] ClientConnection:187 | [<none> -> pulsar://localhost:6650] Create ClientConnection, timeout=10000
2024-07-25 13:25:13.743 INFO  [0x1f6ac8c00] ConnectionPool:124 | Created connection for pulsar://localhost:6650-pulsar://localhost:6650-0
2024-07-25 13:25:13.744 INFO  [0x16d493000] ClientConnection:403 | [[::1]:51066 -> [::1]:6650] Connected to broker
2024-07-25 13:25:13.749 INFO  [0x16d493000] HandlerBase:111 | [persistent://testing-tenant/tnamespace/function-topic, function-sub, 0] Getting connection from pool
2024-07-25 13:25:13.750 INFO  [0x16d493000] BinaryProtoLookupService:86 | Lookup response for persistent://testing-tenant/tnamespace/function-topic, lookup-broker-url pulsar://localhost:6650, from [[::1]:51066 -> [::1]:6650] 
2024-07-25 13:25:13.751 INFO  [0x16d493000] ConsumerImpl:300 | [persistent://testing-tenant/tnamespace/function-topic, function-sub, 0] Created consumer on broker [[::1]:51066 -> [::1]:6650] 
Received message: 'Hello Pulsar 40 EVEN'
Received message: 'Hello Pulsar 41 ODD'
Received message: 'Hello Pulsar 42 EVEN'
Received message: 'Hello Pulsar 43 ODD'
Received message: 'Hello Pulsar 44 EVEN'
Received message: 'Hello Pulsar 45 ODD'
Received message: 'Hello Pulsar 46 EVEN'
Received message: 'Hello Pulsar 47 ODD'
Received message: 'Hello Pulsar 48 EVEN'
Received message: 'Hello Pulsar 49 ODD' 
Delete a function: 

If you want to delete a function we can do that using a pulsar-admin 



/pulsar/bin/pulsar-admin functions delete --tenant testing-tenant --namespace tnamespace --name test-function
The above delete operation will delete a function but it doesnt remove the function metadata. I didnt not find any command to automatically delete the function metadata so for now we have to remove it manually from the below path



rm -rf /pulsar/packages-storage/function/testing-tenant/tnamespace/test-function/*
 

 

