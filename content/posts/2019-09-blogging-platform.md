---
title: "How to set up a Hugo-powered blog platform on AWS"
date: 2019-09-13T17:35:35+02:00
draft: true
description: "This is a multi-post article detailing the process for setting up a Hugo-powered blog platform using AWS services and Gitlab."
---

This is the first of a multi-post article detailing the process for setting up a Hugo-powered blog platform using AWS services and Gitlab.

Hugo is a static site generator written in Go - it takes content written in Markdown format and generates flat HTML which can then be served by any number of web server platforms.

The advantages of using a static site generator over a content management system are speed and flexibility. Flat HTML files can be delivered by a variety of servers ranging from S3 through to a fully-fledged web server like Nginx; and the lack of need for databases and interpreted platforms like PHP or Rails makes setup and configuration much less complicated.

This post series is a complete walkthrough of how to set up a blog using Hugo, AWS and Gitlab. You could substitute another combination of version control and continuous integration service for Gitlab, but this offers a simple CI pipeline that allows deployment in response to merging a pull request.

The process has a number of moving parts:

* [Hugo](https://gohugo.io) to generate the site content
* [Gitlab](https://gitlab.com) to host the source code, and provide the CI workflow for automatic build and deployment
* [S3](https://aws.amazon.com/s3) to store and serve the site itself
* [CloudFront](https://aws.amazon.com/cloudfront) to front S3 to provide a CDN and serve over HTTPS
* [CloudFormation](https://aws.amazon.com/cloudformation) and [Lambda](https://aws.amazon.com/lambda) to provide a workaround for a specific limitation of S3 web serving
* [Certificate Manager](https://aws.amazon.com/certificate-manager) to create SSL certificates to enable serving the site over HTTPS
* [Route53](https://aws.amazon.com/route53) to manage DNS

This looks complicated, but the end result will be a site that can securely served, handle high traffic loads, and be updated with a pull request-based workflow.

# The process

The process is broken down into 6 steps, each of which has a dedicated post. As each post gets written, you can find them from the links below.

* [Setting up Hugo locally, and configuring it as a blogging platform] ( {{< relref "2019-09-setting-up-hugo.md" >}} )
* Setting up Gitlab so that you can use pull requests to trigger builds and deployments
* Setting up your blog's domain with Route53, and using AWS to manage SSL certificates to enable serving the site over HTTPS
* Using S3 as a web server to serve the blog's content
* Setting up Cloudfront to act as a CDN for the blog's content to DDOS-proof your site
* Using AWS Lambda and Cloudformation to rewrite URLS on the fly, working around a couple of S3-and-Hugo-created edge cases

Although it might seem like a lot of steps, the end result is a workflow that will be familiar to anyone working with source code; and a blogging platform that can withstand high traffic loads at minimal cost. It's also a good walkthough some of the many services that AWS offers, and an introduction to some of the processes and terminology that Amazon uses.
