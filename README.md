![FIWARE Banner](https://fiware.github.io/tutorials.Subscriptions/img/fiware.png)

This tutorial teaches FIWARE users about how to create and manage context data subscriptions.

The tutorial builds on the entities and [Stock Management Frontend](https://github.com/Fiware/tutorials.Subscriptions/tree/master/proxy)
application created in the previous [example](https://github.com/Fiware/tutorials.Accessing-Context/) to enable users to understand
the [NGSI](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json)
Subscribe/Notify paradigm and how to use NGSI subscriptions within their own code.

The tutorial refers to Stock Management actions made within the browser combined with  [cUrl](https://ec.haxx.se/) commands. The
cUrl commands are also available as [Postman documentation](http://fiware.github.io/tutorials.Subscriptions/).

[![Run in Postman](https://run.pstmn.io/button.svg)](https://www.getpostman.com/collections/fb5f564d9bc65fc3690e)

# Contents


# Subscribing to Changes of State

> "Don't call us, we'll call you"
>
> — Dorothy Kilgallen (The Voice Of Broadway)


Within the FIWARE platform, an entity represents the state of a physical or conceptural object which exists in the real world. 
Every smart solution needs to know the current state of these object at any given moment in time. 

The context of each of these entities is constantly changing. For example, within the stock management example, the context will
change as new stores open up, products are sold, prices change and so on. For a smart solution based on IoT sensor data, this issue
is even more pressing as the system will constantly be reacting to changes in the real world.

Until now all the operations we have used to change the state of the system have been **synchronous** - changes have been made by 
directly by a user or application and they have been informed of the result. The Orion Context Broker offers also an **asynchronous**
notification mechanism - applications can subscribe to changes of context information so that they can
be informed when something happens. This means the application does not need to continously poll or repeat query requests.

Use of the subscription mechanism will therefore reduce both the volume of requests and amount of data being passed between components
within the system. This reduction in network traffic will improve the overall responsiveness.

## Entities within a stock management system

The relationship between our entities is defined as shown:

![](https://fiware.github.io/tutorials.Subscriptions/img/entities.svg)

The items highlighted in blue are provided by the external context providers. 


## Stock Management Front-End

In the previous [tutorial](https://github.com/Fiware/tutorials.Accessing-Context/), a simple Node.js Express application was created.
This tutorial will use the monitor page to watch the status of recent requests, and a store page to buy products. Once the services
are running these pages can be accessed from the following URLs:

#### Event Montior

The event monitor can be found at: `http://localhost:3000/app/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Subscriptions/img/monitor.png)

#### Store 002

Store002 can be found at: `http://localhost:3000/app/store/urn:ngsi-ld:Store:002`

![Store](https://fiware.github.io/tutorials.Subscriptions/img/store2.png)


# Architecture

This application will make use of only one FIWARE component - the [Orion Context Broker](https://catalogue.fiware.org/enablers/publishsubscribe-context-broker-orion-context-broker). Usage of the Orion Context Broker is sufficient for an application to qualify as *“Powered by FIWARE”*.

Currently, the Orion Context Broker relies on open source [MongoDB](https://www.mongodb.com/) technology to keep
persistence of the context data it holds. To request context data from external sources, a simple Context Provider NGSI 
proxy has also been added. To visualise and interact with the Context we will add a simple Express application 


Therefore, the architecture will consist of four elements:

* The Orion Context Broker server which will receive requests using [NGSI](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json)
* The underlying MongoDB database associated to the Orion Context Broker server
* The Context Provider NGSI proxy which will will:
  + receive requests using [NGSI](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json)
  + makes requests to publicly available data sources using their own APIs in a proprietory format 
  + returns context data back to the Orion Context Broker in [NGSI](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json) format.
* The Stock Management Frontend which will will:
  + Display store information
  + Show which products can be bought at each store
  + Allow users to "buy" products and reduce the stock count.

Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run
from exposed ports. 

![](https://fiware.github.io/tutorials.Subscriptions/img/architecture.svg)

# Prerequisites

## Docker

To keep things simple both components will be run using [Docker](https://www.docker.com). **Docker** is a container technology
which allows to different components isolated into their respective environments. 

* To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
* To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
* To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A 
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml) is used
configure the required services for the application. This means all container sevices can be brought up in a single 
commmand. Docker Compose is installed by default  as part of Docker for Windows and  Docker for Mac, however Linux users 
will need to follow the instructions found  [here](https://docs.docker.com/compose/install/)

## Cygwin 

We will start up our services using a simple bash script. Windows users should download [cygwin](www.cygwin.com) to provide a
command line functionality similar to a Linux distribution on Windows. 

# Start Up

All services can be initialised from the command line by running the bash script provided within the repository:

```console
./services create; ./services start;
```

This command will also import seed data from the previous [Stock Management example](https://github.com/Fiware/tutorials.Context-Providers) on startup.

>:information_source: **Note:** If you want to clean up and start over again you can do so with the following command:
>
>```console
>./services stop
>``` 
>

#  Using Subscriptions

To follow the tutorial correctly please ensure you have the follow pages available on tabs in your browser before you enter any cUrl commands.

#### Event Monitor

The event monitor can be found at: `http://localhost:3000/app/monitor`

#### Stores

The stores can be found at:

* Store 1 -  `http://localhost:3000/app/store/urn:ngsi-ld:Store:001`
* Store 2 -  `http://localhost:3000/app/store/urn:ngsi-ld:Store:002`


## Setting up a simple Subscription 

Within the stock management example, imagine that the regional manager of the company wants to alter the price of a product. 
The new price should immediately be reflected at the till in all stores within the system. It would be possible to set up the
system so that it was constantly polling for new information, however prices are not changed very frequently so this would be
a waste of resources and create a lot of unnecessary data traffic.

The alternative is to create a subscription which will POST a payload to a "well-known" URL whenever a price has changed. 
A new subscription can be added by making a POST request to the `/v2/subscriptions/` endpoint as shown below:

#### Request:

```console
curl --request POST \
  --url 'http://{{orion}}/v2/subscriptions/' \
  --header 'content-type: application/json' \
  --data '{
  "description": "Notify me of all product price changes",
  "subject": {
    "entities": [
      {
        "idPattern": ".*", "type": "Product"
      }
    ],
     "condition": {
      "attrs": [ "price" ]
    }
  },
  "notification": {
    "http": {
      "url": "http://context-provider:3000/subscription/price-change"
    }
  }
}'
```


The body of the POST request consists of two parts, the `subject` section of the request (consisting of `entities` and `conditions`)states that the subscription will be fired whenever the `price` attribute of any **Product** entity is altered. The notification section of the body states that once the conditions of the subscription have been met, a POST request containing all affected **Product** entities will be sent to the URL `http://context-provider:3000/subscription/price-change` which is handled by the stock management front-end application.

For a first run, when the subscription is created, the Orion Context Broker runs the `condition` test, and since it has not been run before makes the assumption that all products have been changed. Therefore a request is sent to `subscription/price-change` immediately as shown:

#### `http://localhost:3000/app/monitor`

![](https://fiware.github.io/tutorials.Subscriptions/img/products-subscription.png)

Code within the Stock Management Front-End application handles received the POST request as shown:

```javascript
router.post('/subscription/:type', (req, res) => {
    _.forEach(req.body.data, item => {
        broadcastEvents(req, item, ['refStore', 'refProduct', 'refShelf', 'type']);
    });
    res.status(204).send();
});

function broadcastEvents(req, item, types) {
    const message = req.params.type + ' received';
    _.forEach(types, type => {
        if (item[type]) {
            req.app.get('io').emit(item[type], message);
        }
    });
}
```

This business logic emits socket io events to any registered parties (such as the cash till)

The cash till has been set to reload if is receives an event - however in this case the prices have not changed yet, so the product prices
remain the same - for example a bottle of beer remains at 0.99€

#### `http://localhost:3000/app/store/urn:ngsi-ld:Store:002`

![](https://fiware.github.io/tutorials.Subscriptions/img/beer-99.png)

Let's reduce the price of a bottle of beer to 0.89€. This can't be done programmatically yet, so it has to be done with a curl command
as shown:


```console
curl --request PUT \
  --url 'http://{{orion}}/v2/entities/urn:ngsi-ld:Product:001/attrs/price/value' \
  --header 'Content-Type: text/plain' \
  --data 89
```

Whenever an attribute of the **Product** entity is updated, the Orion Context Broker checks for any existing subscriptions 
(which exist for that entity) and  applies the `condition` test. This time only one **Product** entity has changed since the last run therefore a POST request is sent to `subscription/price-change` - which only contains one **Product** in the body:

#### `http://localhost:3000/app/monitor`

![](https://fiware.github.io/tutorials.Subscriptions/img/price-change.png)

This business logic  on the Stock Management Front End again emits socket io events to any registered parties (such as the cash till)
and since the price has changed the till now displays  a bottle of beer remains at 0.89€

#### `http://localhost:3000/app/store/urn:ngsi-ld:Store:002`

![](https://fiware.github.io/tutorials.Subscriptions/img/beer-89.png)   
