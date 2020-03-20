---
title: "API Gateways"
date: 2020-03-17T13:15:48+01:00
draft: false
summary: Everything you need to know about API Gateways, and a market overview of existing products. This is a post about a talk that was given recently at the finleap Engineering Meetup.
author: Sebastian Weikart
author_position: Engineering Lead
author_email: sebastian.weikart@finleap.com
---

At **finleap**, we recently started a monthly gathering of engineers across the ecosystem. 

These includes people from **finleap Build**, the original company builder, as well as our very own financial Software-as-a-Service company [finleap Connect](https://connect.finleap.com/), and all the portfolio companies such as [Element](https://www.element.in/), [solarisBank](https://www.solarisbank.com/), [Penta](https://getpenta.com/), [Elinvar](https://elinvar.de/), [Clark](https://www.clark.de/), [Joonko](https://joonko.de/) and more, as well as newly hatched ventures that are building new products from scratch. 

All of those companies come with their own set of challenges, be it how to hit the market as fast as possible with a high quality product, or operate and run a financial platform with extremely high levels of availability and inpenetrable security.

If you want to be part of this, you can find open tech positions across the whole portfolio here: <https://www.finleap.com/careers/tech/>

It's a great opportunity to learn from each other, understand how these challenges were mastered in the past, and share a beer and a laugh together.

At the last meetup in March, I gave a talk about **API Gateways** - a common pattern about what they are, where they came from, and a short and totally subjective overview of the available solutions on the market. I came across this pattern and technology in 2014, and was interested in it ever since, following the different development and products that hit the market.

In this post I will share my thoughts and the presentation. Please note that some statements are my personal opinions and based on experiences I made in my past career. If you want to discuss them with me, feel free to get in touch.

## API Gateways - What are They?

It all started with an architecture pattern that turned into really powerful tools and a whole industry of products out there. Among the abilities are the following:

- It’s an **Edge Service** – i.e. can sit on the edge of your network
- It’s a **Proxy** - it serves as a single access point, exposing a single address to the clients
- It’s a **Reverse Proxy** - it is usually deployed infront of many other services
- It’s a **Level 7 load balancer** – or even more than that as it can make routing decisions not only based on HTTP headers or content, but also based on custom logic
- It’s a **Router** - it can decide where to route a request to. A client doesn't need to know which service exactly is being addressed
- It provides **Access Control** - one of the key abilities is to allow or prevent access to your APIs
- It can **translate protocols and formats** - while outside clients most likely use HTTP and JSON, inside in your system you might use different protocols and formats like maybe Protocol Buffers or Messaging Systems
- It takes care of common functionality and help **separate concerns** in your architecture
- It can be an **API Orchestrator** - you can orchestrate different APIs into one single API endpoint
- It can help to slice and dice a portfolio of APIs to better suit certain use cases (ex. **Backend-for-Frontend**)
- Provide an **API Portal**
- The silver bullet for all your problems (Quote: Vendor XYZ)

## Common Patterns

### In The "Olden Days"

API Gateways as an architecture pattern was canonized with the arrival of microservices architecture, but it was a well-known technique long before that.

With the arrival of mobile and Ajax, as well as REST becoming fashionable, an API gateway was often built a separate application that was forwarding API calls from the outside. Maybe you mapped to a legacy SOAP API or to another legacy stack.

![API Gateway The History](/apigateway/api-gateway-scenarios-Page-1.png "API Gateways - The History")

### Gateway for Message-Oriented, RPC or CQRS Architectures

Let's say our backend doesn’t follow a REST architecture. We still need to provide an API for outside clients to connect. We can use asynchronous designs in the backend such as RPC or CQRS using protocols such as JMS, ActiveMQ, RabbitMQ, or Protocol Buffers or we have a "Service Bus" and messages need go into the bus from the outside. 

In this case, putting an API Service, aka API Gateway, in front will do the job, and it will also give us the flexibility to add REST-style resources. Some products we will discuss later come with adapters for common protocols such as for Kafka, AMQP and more.

![The RPC or CQRS Gateway](/apigateway/api-gateway-scenarios-Page-2.png "The RPC or CQRS Gateway")

### The Indespensible Ingredient in a Microservices Architecture

In the "microservice age", we built lots of services with their own API. Suddenly, a client needed to know about all those services, needed to authenticate for each one separately, and potentially speak a different protocol or format. 

API Gateway was the obvious pattern we needed to cover common functions, unify the interface and add a routing layer so the frontend doesn't need to know the address and specification of every single backend service. API Gateways have been extended to cover functionality such as authentication / authorisation, routing and service discovery.

Effectively, the API Gateway allows a separation of purely technical concerns and business concerns.

![The Indespensible Ingredient in a Microservices Architecture](/apigateway/api-gateway-scenarios-Page-3.png "The Indespensible Ingredient in a Microservices Architecture")

### The Hipster Variant - GraphQL

The latest API technology - GraphQL - is a naturally good fit to live inside an API Gateway. To adopt GraphQL into an existing infrastructure, you either need to build it directly into your services, or build an intermediate layer that resolves a GraphQL query to different data sources (also called API Orchestration in the REST API context) – a **GraphQL Gateway**.

This allows federation of multiple data sources into a single API, and a new variation of BFF pattern. Frontend developers can slice and dice the API formats directly to their needs using GraphQL queries.

![API Gatway with GraphQL](/apigateway/api-gateway-scenarios-Page-4.png "API Gatway with GraphQL")

### Backend for Frontends (BFF)

This brings us neatly to the backend for frontends pattern. This pattern was developed from the need to scale-out teams. Suddenly, a centralised API Gateway became rather unwieldy and created a strong dependency on a team that took care of it. 

To mitigate that, it is a good practice to actually allow multiple API Gateways, which are within the responsibility of an individual team, serving their own very special needs. So instead of having one API to rule them all, give the teams the tools to be masters of their own destiny.

![API Gatway Backend for Frontends](/apigateway/api-gateway-scenarios-bff.png "API Gatway Backend for Frontends")

### Orchestrator Services

Another problem teams will face once they scale is that the API Gateway has been "overloaded" with a lot of functionality, maybe even custom logic. While this is of course one of the possible functions that an API gateway can bring to the table, it suddenly creates new dependencies again, and perfectly-autonomous teams become reliant on each other.

For example, change cycles, configuration and compatibility problems will inhibit the ability to develop new services and features.

The solution to this is: Simply build the orchestrator service you need separately, and externalise it as a new, separate service. Job done.

![Orchestrator Services](/apigateway/api-gateway-scenarios-Page-5.png "Orchestrator Services")

### Put a Load Balancer In Front

A successful solution using API Gateway will soon reach the point where it needs to scale the API gateway itself. While some API Gateways have been built on top of powerful load balancers and include features like TLS termination, it might make sense to have a dedicated load balancer to do the job, as you scale out the load balancer as part of your application scale set.

![Scaling Out](/apigateway/api-gateway-scenarios-Page-7.png "Scaling out")

### Anti-Patterns

Here is a short list of patterns that can reduce benefits you can draw from an API Gateway. In the end, every architecture with API Gateway will gravitate towards a high degree of centralisation, in which lies the greatest peril.

- **Overambitious API Gateway:** Refrain from tendencies that lead to dangerous concentrations of centralization – and to loss of flexibility therefore due to more complex change and deployment procedures. Don’t put too much logic, orchestration etc into the API Gateway – this again leads to hazardous levels of centralization. As we say, BFFs are your BFF.
- **Single Point of Failure:** It is easy to see how an API Gateway becomes a single point of failure. explore ways how to scale out early. Ideally keep it stateless. Some API Gateways have a very neat scaling model.
- **Fake Service Mesh:** Don’t put an API Gateway in front of every service – this will lead to a counterfeit service mesh, and it will get expensive as you consume more compute resources

## API Gateway - Market Overview

Here is a list of features and properties you should consider then evaluating API Gateway products:

- Easy scale out model, ideally stateless
- Declarative way of defining resources and routes. If it's declarative, it will fit very nicely in an Infrastructure-as-Code approach and the configuration and routes can be templated
- Support for open standards such as OpenAPI
- Integrate with 3rd party monitoring
- Authentication
- Authorisation
- Access control
- Load balancing
- Routing
- Rate limiting
- Caching
- Manage API clients
- Metrics collection
- Request logging
- Health checks
- Circuit breaker
- Support for your protocols and formats
- Plugin architecture / custom adapters
- Add CORS Headers
- Simple orchestration of backend services

Some products are more like a toolbox or a framework, where you can pick the functions you need, and maybe add your own, others are more like a swiss army knife, giving you all sorts of implements.

For the following market overview, I deliberately focus on products that are available for free in some shape or form. Keep in mind that every cloud vendor has their own managed API Gateway readily available.

If you are building based on those clouds, you probably want to evaluate them first, because they are usually very well and tightly integrated with other services inside the cloud vendor platform. Other than that, there exist a few other enterprise level products.

When evaluating, I will give a short and completely subjective verdict from my perspective. If I give a negative conclusion, it doesn't mean that the product is still a very good fit for your specific usecase.

### Commercial Products

#### Cloud Provider Gateways

- Amazon API Gateway: <https://aws.amazon.com/api-gateway/>
- APIGee (now Google Cloud) - <https://cloud.google.com/apigee>
- Google Cloud Endpoints - Google actually has another API Gateway Product: <https://cloud.google.com/endpoints>
- Azure API Gateway: <https://azure.microsoft.com/en-us/services/api-management/>

#### Enterprise Vendors

- Dell Boomi: <https://boomi.com/>
- Mulesoft: <https://www.mulesoft.com/>
- 3Scale (Redhat / IBM): <https://www.3scale.net/>
- Axway: <https://www.axway.com/>

They all fall firmly in the (expensive) Swiss Army Knife Category.

### Spring Cloud Gateway

[Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway) was the very first real API Gateway framework, and had its roots in the famous Netflix Microservices Stack. It's more a toolbox / framework then a standalone product.

#### The Good

- Proven Java Technology and APIs
- Very easy to add in your Java Spring Project

#### The Bad

- It‘s more a framework then a fully fledged product (maybe that‘s not so bad ;)
- All the configuration is in Java code. Spring used to have declarative XML files which have been ditched in favour for in-line Java fode annotations. Which makes sort of sense in the Java world.

#### Conclusion

Conclusion: Great toolbox and best option to use if you are firmly located within the Spring world.

### Kong

[Kong](https://konghq.com/) is based on proven nginx technology, and open source since 2015, therefore one of the first products in the field. It falls into the Swiss Army Knife Category.

#### The Good

- Proven base technology nginx
- One of the oldest products in the field
- Everything can be controlled through Control API
- Cloud Function Adapters available

#### The Bad

- Questionable scale out model – needs Postgres or Cassandra
- Control API is imperative
- Declarative configuration added as an afterthought and mixed up with database-less operation model, makes infrastructure-as-Code cumbersome
- Need to pay for UI
- Need to pay for a lot of features
- Custom plugins written in Lua – a rather obscure language for me

#### Conclusion

Hasn‘t aged well in my opinion – complex scaling and operating model, need paid tier fast. Declarative model was added as an afterthought.

### Tyk

[Tyk](https://tyk.io/) was shortly afoot after Kong was released. It is built from scratch in Go and offers a management UI and analytics out of the box. The company offers a managed service. It falls into the Swiss Army Knife Category.

#### The Good

- Everything can be controled through a Control API
- UI included field
- Built in Go
- You can develop Plugins in Go and Javascript (JSVM)

#### The Bad

- Requires a Redis to run and potentially a MongoDB for Dashboard
- Not built „declarative first“, makes Infrastructure-as-Code cumbersome
- Scaling out requires paid version (i.e. >1 node)
- Something like Tyk Pump between „Dashboard“ and Gateway – complex setup
- No protocol transformation out of the box

#### Conclusion

Gets expensive fast as you need to pay once you scale out, full operating model is complex.

### Express Gateway

Based on NodeJS and the proven Express Framework, [Express Gateway](https://www.express-gateway.io/) is one of the newer entrants in the API Gateway Arena.

#### The Good

- Proven technology NodeJS and Express
- Build declarative first
- It‘s just JavaScript
- Easy to customise
- No need for a database – completely stateless and therefore easy scale out model

#### The Bad

- It‘s JavaScript – I guess that can put some people off
- Relatively new technology

#### Conclusion

A real option, easy to customise and build your own gateway in JavaScript / Node, easy to scale out.

### KrakenD

A new, very nimble option build from scratch in Go is [KrakenD](https://www.krakend.io/).

#### The Good

- Needs no database – completely stateless, scales very well horizontally
- Completely declarative
- Got some nice adapters, called Backends, for RabbitMQ, Kafka, AWS Lambdas and more
- It doesn‘t do stuff other systems do really well – like Monitoring and Service Discovery. Instead - it pre-integrates with standard solutions
- Benchmarks extremely well compared with Kong and Tyk
- Can aggregate multiple API Calls

#### The Bad

- Relatively new technology
- Custom enhancements via Lua and Go

#### Conclusion

Yes it‘s a Swiss Army Knife, but more of the small, practical sort. The declarative approach and the easy scale model is a big plus, and the adapters for different backend protocols give great flexibility to integrate it in existing architectures.

### Ambassador

[Ambassador](https://www.getambassador.io/) is a new player, based on Envoy Proxy, exclusive to Kubernetes

#### The Good

- Completely declarative configuration
- Based on Envoy means it has already great support for advanced features like canary routing and shadowing
- Uses K8S-native capabilities for load balancing, service discovery and more
- Pre-integrates with standard solutions for monitoring

#### The Bad

- It looks like they are trying to build another one-size-fits-all solution
- Runs only in Kubernetes

#### Conclusion

Powerful reverse proxy with added functionality, worth trying if you run K8S.

### Apollo Gateway

[Apollo Gateway](https://www.apollographql.com/docs/apollo-server/api/apollo-gateway/) is a GraphQL gateway to federate other GraphQL APIs

#### The Good

- More of a framework kind of thing, designed to build your own custom gateway
- Made by early GraphQL adopters

#### The Bad

- Only GraphQL backends supported
- In practise, federating GraphQL APIs can turn out very complex

#### Conclusion

Only for niche use-cases when you want / need to federate other GraphQL APIs.

### Sqoop

Based on Envoy Proxy just like Ambassador, [Sqoop](https://sqoop.solo.io/) it is exclusive to Kubernetes. It is an attempt to federate all sorts of APIs into a GraphQL API.

#### The Good

- Adapters available for gRPC, Serverless – not just REST or GraphQL
- Based on Envoy means it has already great support for advanced features like canary routing and shadowing
- Uses K8S-native capabilities for load balancing, service discovery and more
- Pre-integrates with standard solutions for monitoring

#### The Bad

- Only for Kubernetes
- I don‘t see how this can work purely with configuration and Go templates
- Very new technology

#### Conclusion

I don‘t have an opinion yet – worth trying if you run K8S and want to have GraphQL

## The Presentation

You can find the talk presentation here:  
{{< pdf src="/pdf/finleap_talk_api_gateway.pdf" width="640" height="420" >}}

## The End

That concludes my API Gateway overview. Please note, a lot of things written above are my personal opinions. If you have questions or comments, feel free to get in touch with me.
