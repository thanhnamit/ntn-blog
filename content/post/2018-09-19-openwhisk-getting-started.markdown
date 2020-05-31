---
featured: false
title: "Implementing the Event-driven Pattern with Openwhisk FaaS"
tags: [serverless, faas, event-driven, openwhisk]
date: 2018-09-19T14:24:10+10:00
lastmod: 2018-09-19T14:24:10+10:00
---

The advent of cloud computing and everything as a service changes the IT landscape significantly. Among these new technologies and paradigms, the Function as a service (FaaS) model has emerged as a new deployment approach that challenges the traditional ones both in technical and business aspects. Leveraging huge cloud platform capacity and elasticity, FaaS can power APIs, event processing, batch jobs, IoT event streams or act as a mediator for backend services and APIs. This model encourages businesses to develop innovative solutions without much up-front investment in infrastructure, it also brings great flexibility and speed for developers as they can choose different programming languages and focus on business logic rather than on provisioning and managing servers. You may find the term `Serverless` is being used more broadly on the Internet, however in this article we prefer FaaS.

Toward the end of 2018, large cloud vendors all have FaaS portfolios including AWS Lambda, Azure cloud functions, Google cloud function and IBM function. Customers who have invested in those cloud vendors can leverage the existing ones. However, for organisations relying on multi-cloud architecture or hybrid IT infrastructure, having the choice of using an open source, vendor-agnostic FaaS platform would be a sensible approach. This article focuses on Openwhisk, an open source FaaS platform which provides a foundation for self-hosted FaaS implementation.

Openwhisk was created by IBM researchers and has gained significant community interests recently. The platform powers the IBM function and has been selected by both Red Hat and Adobe as the FaaS component for their cloud offerings. We will explore how Openwhisk can be used to implement an event-driven architecture for an online business - Lucky book store.

### Lucky book store problem
Lucky bookstore wants to revamp their e-commerce presence prior to Xmas to handle increasing order volume. The main requirement is to reimplement their website and other backend components such as loyalty and inventory services so the business can provide timely communication to their loyal customers while cutting infrastructure cost. The new IT architects mandate that they want to completely rebuild their monolith stack to replace with microservices and APIs. After a lot of spike meetings, we end up with the new architecture built on Openwhisk FaaS platform.

![New design](/img/posts/2018-10-19-openwhisk-design.png 'Design')

This design has following components:

**Order API**: accept orders sent by frontend apps. Upon receiving a valid message, it dispatches the order to a message broker and return an acknowledgment to caller. This api is implemented as a function with Openwhisk.

**Message broker**: deploy Apache Kafka broker to serve as pub/sub component.

**Loyalty service**: subscribe to order service topic and update loyalty points for customer, it passes the result to notification service if the customer needs to be notified.

**Notification service**: notify customers via SMS or email as specified by loyalty service.

**Inventory service**: subscribe to order service topic and integrate with Warehouse system and ERP.

The implication of this design is that all microservices in the diagram above are scalable as they are implemented as Openwhisk functions, they are only activated when orders are submitted from frontend apps. The design also allows functions to scale to cope with spikes of large order volume during sales seasons. This satisfies the initial requirements for the business.

### Install Openwhisk and CLI tools
To get the Openwhisk up and running, there are several options, if you are fan of IBM or willing to experiment without troubles, you can register an account on [IBM Bluemix](https://console.bluemix.net/openwhisk/) and use Openwhisk IBM Function on their free tier. Alternatively, you can run Openwhisk on [Kubernetes](https://github.com/apache/incubator-openwhisk-deploy-kube#setting-up-kubernetes-and-helm) or [Red Hat OpenShift](https://github.com/projectodd/openwhisk-OpenShift). In this blog, we demonstrate deploying Openwhisk with Docker in your local machine, runnable source code and instructions can be found at [openwhisk-luckystore](https://github.com/nnguyen-dpe/openwhisk-luckystore). The file `OPENWHISK.md` contains instructions to configure your local environment.

### Part I - Implement the order service

In this part, we illustrate how to deploy the order api as a REST API. Openwhisk action (or F in FaaS) is the core component that can be invoked by triggers. An action executes the provided logic and returns output. Actions can be implemented in languages and runtimes of choice; the developers have to provide the code, its dependencies and optionally specify max execution time and memory. Once the action is deployed, it can be invoked in two styles: blocking and non-blocking. A blocking invocation waits for the completion of the function and return the result to the caller. For non-blocking one, the caller is provided with an activation id so the result can be fetched from the platform later. This asynchronous mechanism improves platform throughput and enables Openwhisk to chain multiple actions together, providing powerful functions composition capability. We examine this feature at the end of the post.

Now for requirement of the API, we want to return a confirmation to the caller, hence the API is deployed as a blocking function. Let's deploy and test the action and examine how Openwhisk handles the request with IBM Cloud Shell tool. For this implementation, we use Python3 and the Flask framework. The following script packages the api with dependencies then creates a function with specific resource limits. You can either run commands line by line or just execute `./order-api/create-func.sh` script.

```sh
cd order-api
# build dependencies with openwhisk python runtime
docker run --rm -v "$PWD:/tmp" openwhisk/python3action bash \
  -c "cd tmp && virtualenv virtualenv --python=python3 && source virtualenv/bin/activate && pip install -r requirements.txt"

# pack for deployment
zip -r order-api.zip __main__.py app.py apis flaskwsk virtualenv requirements.txt README.md

# create action with execution time of 500 millis & memory of 128MB
wsk -i action create order-api --kind python:3 order-api.zip --web raw -t 500 -m 128

# get callable action url
wsk -i action get order-api --url

# call the health endpoint
curl -k https://{WHISK.HOST}/api/v1/web/guest/default/order-api/api/v1/health
```

This API is exposed without going through an API gateway. In production, it is required to deploy a full-scale API management. To accommodate this, Openwhisk does ship with a default API gateway or you can plug a third party API gateway such as Kong, we omit this part for sake of simplicity. To test the API, invoke the following script.

```sh
./order-api/invoke-func.sh {WHISK.HOST}
```

Let's verify the function is successfully called. As you see below, Openwhisk captures details of each invocation in an activation record, this record is stored in a high performant NoSQL engine CouchDB, in localhost you browse its data by navigating to [CouchDB](http://localhost:5984/_utils). Some key information are response status, start, end, logs and result.

```sh
# activation list command returns list of activation with most recent one on top
wsk -i activation list --limit 1 order-api

# get details by activation id
wsk -i activation get {activation_id}
```


### Part II - Publish/Subscribe message via Kafka message broker

Apache Kafka is a messaging broker we use to create an event backbone. We choose Kafka due to its high scalability, high throughput and fault tolerant design over traditional JMS/AMQP implementations. Kafka leverages distributed, sequential log-structured approach to store messages in disk buffer which improves reliability while maintaining high performance either in reading or writing messages. It is interesting fact that Openwhisk platform is built upon Kafka message broker at its heart. Back to our implementation of the order system, without going into too much detail around how to config a Kafka broker, we can resuse Openwhisk Kafka instance as the message broker. Run the following command to create a new topic called `order-service-topic` in Kafka via a docker container provided by Confluent.io.

```sh
docker run \
  --net=openwhisk_default \
  --rm confluentinc/cp-kafka:4.1.2-2 \
  kafka-topics --create --topic order-service-topic --partitions 1 --replication-factor 1 \
  --if-not-exists --zookeeper zookeeper:2181
```

Now we need to update the order API so it can send message to the topic. Let update the class OrderService `order-api/apis/service.py`. If you don't want to update code, please checkout the branch feature/part2 in openwhisk-luckystore.

```python
import uuid
import logging
import json
from kafka import KafkaProducer

log = logging.getLogger(__name__)

class OrderService(object):

    def __init__(self):
        self.__topic = "order-service-topic"
        self.__producer = KafkaProducer(
            bootstrap_servers='kafka:9092',
            value_serializer=lambda v: json.dumps(v).encode('utf-8'))

    def create(self, data):
        data['id'] = str(uuid.uuid4())
        self.__producer.send(self.__topic, data)
        return data['id']

_srv = OrderService()
```

Redeploy the function is a straightforward task by running the command.

```sh
./order-api/update-func.sh
```

Let test the API and browse kafka topics to verify if order messages are landed successfully on the topic.

```sh
./order-api/invoke-func.sh {WHISK.HOST}
```

To view the order message, open the Kafka UI topic browser by visiting `http://localhost:8001`


### Part III - Implement and Compose the Backend Services

With Kafka as event source, we can now deploy other backend services by connecting actions to the source. In order to establish the connection, we need to define a `feed`, `trigger` and `rule`. A rule tells Openwhisk which action to be invoked when the trigger is fired, a trigger is a channel for same class of event. The trigger then can be activated by an external event source which is configured as a feed. Openwhisk provides a Kafka package with ready-to-use Kafka feed and actions for publishing and consuming messages. You can verify if Kafka feed is available with the following command.

```sh
wsk -i package list /whisk.system
wsk -i package get --summary /whisk.system/messaging
```

```sh
wsk -i trigger create OrderKafkaTrigger -f /whisk.system/messaging/kafkaFeed -p brokers "[\"kafka:9092\"]" -p topic order-service-topic
```

We then create an action called `loyalty-service` which is the first consumer of the topic. This service gets a message from the topic and updates customer loyalty points.

```sh
./loyalty-service/create-func.sh
```

We complete this by linking the action to the trigger via a rule.

```sh
wsk -i rule create OrderEventRule OrderKafkaTrigger loyalty-service
```

Let's execute an end to end test and verify the result by examining the loyalty-service result.

```sh
# invoke
./order-api/invoke-func.sh {WHISK.HOST}

# get activation id
wsk -i activation list --limit 1 loyalty-service

# check activatio log
wsk -i activation result {activation_id}
```

In practical, a microservice can act as a controller or interactor which orchestrates other services to fulfill a specific business rule. In our scenario, we can link the `loyalty-service` with `notification-service` so that when loyalty point is updated, `notification-service` is invoked to send out a message to customer. Openwhisk offers a programming model called composer to accomplish this goal. A [composer](https://github.com/apache/incubator-openwhisk-composer) defines a workflow orchestration and wrap it in a function. Firstly, let's create `notification-service`

```sh
./notification-service/create-func.sh
```

To use composer, we need to install a Node.js package.

```sh
npm install -g @ibm-functions/composer
```

We create a javascript file called `loyalty-rule.js` with following content to specify the logic of the composition. What this function does is to invoke `loyalty-service`, check if outcome is `true` then calls `notification-service` with same params, otherwise calls system's `echo` function.

```javascript
const composer = require('@ibm-functions/composer')

module.exports = composer.if(
    composer.action('/guest/loyalty-service'),
    composer.action('/guest/notification-service'),
    composer.action('/whisk.system/utils/echo')
)
```

Now deploy the new function with the following commands.

```sh
compose loyalty-rule.js > loyalty-rule.json
deploy loyaltyrule loyalty-rule.json -w -i
```

If you use IBM Cloud Shell tools, you can view the loyaltyrule by typing `app get "/guest/loyaltyrule"`, as illustrated below.

![Service composition](/img/posts/2018-10-19-openwhisk-service-composition.png 'Service composition')

Finally, let's rerun the full end to end transaction and check the results.

```sh
# open a new terminal and poll activation result
wsk -i activation poll
```

```sh
# invoke the api
./order-api/invoke-func.sh {WHISK.HOST}
```

As the activation result indicates, the function triggered by loyaltyrule invoked other functions according to the composition rule, we also see Openwhisk wraps all related activation ids for the parent invocation, which helps debugging easier.

### Part IV - Packaging Openwhisk artefacts

Openwhisk enables developer to share their work via packages. A package is a bundle of related actions and feeds which are reusable components. Packages also allow sharing common parameters among those entities, best practices for creating namespaces in programming or reusable libraries can be applied to Openwhisk package. For our design, because both `loyalty-service` and `inventory-service` are consuming same order event type from Kafka, it makes sense to bundle them together as a package. The package then can be shared so other teams can add relevant functions or services to it.

```sh
# create a package which bundle loyalty-service and inventory-service
wsk -i create package order-handlers --shared yes

# recreate actions in the package
wsk -i action create /guest/order-handlers/loyalty-service --kind python:3 loyalty-service.zip
wsk -i action create /guest/order-handlers/inventory-service --kind python:3 inventory-service.zip

# view package content
wsk -i package get --summary /guest/order-handlers
```

### Final words

FaaS platforms are relatively new technology, as such the development and operation tools are somewhat limited, for example developers often find it hard to debug FaaS without using some vendors' tools. With open source FaaS and container technologies, developers are capable to deploy and test entire platform locally. The Openwhisk community has developed numerous tools and reusable integrations with other cloud services. The [Openwhisk official documentation](https://openwhisk.apache.org/documentation.html) provides full list of supported integrations and tools.

Any FaaS platform has its own resource limitations due to the multi-tenant architecture and Openwhisk is no exception. By default there are limitations on execution time and memory, but those are configurable depend on capacity of the infrastructure and initial design. These limitations also can be addressed by deploying Openwhisk on top of platforms such as Kubernetes or OpenShift, which address scalability at a cluster level. Developers need to be aware of those limitations and factor in the design and implementation of the function.

Openwhisk offers a unique value proposition as being fully open source FaaS platform, providing a supple deployment model and out of the box scalability. The platform supports a rich and open ecosystem for collaborating microservices and sharing packages. At this time, Openwhisk is a leading open source FaaS platform due to its vibrant community and supports from vendors such as IBM, Red Hat and Adobe.

