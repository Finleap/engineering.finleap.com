---
title: "API Gateways"
date: 2020-03-17T13:15:48+01:00
draft: true
summary: Explaining API Gateways and giving a market overview of existing products. This is a post about a talk that was given recently at the Finleap Engineering Meetup.
author: Sebastian Weikart
author_position: Engineering Lead
author_email: sebastian.weikart@finleap.com
---

At **finleap**, we recently started a monthly gathering of engineers across the ecosystem, which includes people from **Finleap Build**, the original company builder, as well as our very own financial Software-as-a-Service company **Finleap Connect**, and all the portfolio companies such as **Element**, **solarisBank**, **Penta**, **Perseus**, **Elinvar**, **Clark**, **Joonko** and more, as well as newly hatched ventures that are building new products from scratch. All of those companies come with their own set of challenges, be it how to hit the market as fast as possible with a high quality product, or operate and run a financial platform with extremely high levels of availability and inpenetrable security.
It's a great opportunity to learn from each other, understand how these challenges were mastered in the past, and share a beer and a laugh together.

At the last meetup in March, I gave a talk about **API Gateways** - a common pattern we what they are, where they came from, and a short and totally subjective overview of the available solutions on the market. In this post I will share my thoughts and the presentation.

## API Gateways - What are They?

It all started with an architecture pattern that turned into really powerful tools and a whole industry of products out there. Among the abilities are the following:

- It’s an Edge Service – i.e. can sit on the edge of your network
- It’s a Proxy - it serves as a single access point, exposing a single address to the clients
- It’s a Reverse Proxy - it is usually deployed infront of many other services
- It’s a Level 7 load balancer – or even more than that as it can make routing decisions not only based on HTTP headers or content, but also based on custom logic
- It’s a Router - it can decide where to route a request to. A client doesn't need to know which service exactly is being addressed
- It provides Access Control - one of the key
- It can translate protocols and formats
- It takes care of common functionality and help separate concerns in your architecture
- It can be an API Orchestrator
- It can help to slice and dice a portfolio of APIs to better suit certain use cases (ex. Backend-for-Frontend).
  Provide an API Portal
  The silver bullet that solves all your problems (Quote: Vendor XYZ)

API Gateways as an architecture pattern was canonized with the arrival of microservices architecture, but it was a well known technique long before that.

## API Gateway - Alternatives

## The Presentation

You can find the presentation here:
{{< pdf src="/pdf/finleap_talk_api_gateway.pdf"}}
