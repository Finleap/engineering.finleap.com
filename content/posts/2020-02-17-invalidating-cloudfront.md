---
title: "Automating Cloudfront invalidations in an continous deployment setup"
date: 2020-02-17T13:15:04+01:00
draft: false
summary: "How to automate the process of invalidating content in Cloudfront distributions, as part of a continuous deployment setup"
---

In [this earlier post](https://finleap.tech/posts/web-serving-on-aws/), I worked through the process of automating deployment of a Hugo blog to an Amazon AWS environment. Part of that included setting up a Cloudfront distribution to act as a content distribution network in front of the S3 bucket.  

Using Cloudfront as a CDN means that the actual content is replicated out into the CDN's edge nodes, and will be served from there rather than hitting the "real" backend. The tradeoff is that changes to the "real" content will take time to replicate out to the edge nodes, so it can be a while before they show up. Not ideal if you want to get new or updated content out there quickly.

A workaround for this delay is to create an _"invalidation"_ - basically telling Cloudfront that the cached versions of specific files are now stale, and need to be updated.

There's a cost for creating invalidations, so they're not something that you want to do multiple times an hour - but for getting the latest version of a site out there, it's not too onerous.

You can create invalidations manually, but a more robust process (and prompted by the fact that I _always_ forget to trigger them manually, and then sit around wondering why nothing has changed) is to add them to the deployment stage of your CI workflow.

### Prerequisites

The prerequisites are:

* you've created a Cloudfront distribution, and you've got a note of its ID
* the AWS user that runs the deployment stage has the `CloudFrontFullAccess` permission policy attached
* you've got a CI/CD workflow set up on Gitlab as per the [original post](https://finleap.tech/posts/web-serving-on-aws/). The basic principles will also apply to other CI services such as Github actions.

### The overall process

The overall process is as follows:

* you push changes to Gitlab, which triggers the build/deploy pipeline
* the _build_ job handles building the files to be pushed to AWS
* the _deploy_ job copies those built files up to S3
* at the end of the _deploy_ job, it creates a Cloudfront invalidation with the AWS CLI tool that's running in the deployment container. That invalidation will force the new and updated files to be replicated immediately.

### Configuring the Gitlab settings

To keep the deployment configuration and the parameters (such as access keys, secrets and Cloudfront distribution ID) separate, we'll use a Gitlab environment variable which will be substituted into the config at runtime.

In the Gitlab project, select the `CI/CD` option from the `Settings` area in the left-hand sidebar, and add a new `Variable`. The `Key` is up to you - I use `PROD_CLOUDFRONT_DIST`. The `value` is the Cloudfront distribution ID from the prerequisites.

![distribution id variable](/cloudfront/dist_id.png "Distribution ID variable")

### Updating the jobs

Once you've got the Cloudfront distribution ID saved as a variable, you can use it in the `.gitlab-ci.yml` configuration file.

In the `prod_deploy` stage we're going to add a third script step, which will create a new Cloudfront invalidation as soon as the built files have been copied to S3.

Add the following below the S3 script:

`aws cloudfront create-invalidation --distribution-id $PROD_CLOUDFRONT_DIST --paths "/*"`

This creates an invalidation for the distribution ID that's stored in the `$PROD_CLOUDFRONT_DIST` variable, and uses a _wildcard path_ to invalidate _all_ files in the distribution.

(Technically, you could scope the wildcard down a bit to save on potential costs (you get 1000 file invalidations free per month, and they cost $0.005 thereafter) - but using the `*` wildcard means that you won't accidentally miss less-obvious changes.)

The complete stage should now look like:

```
prod_deploy:
  stage: prod_deploy
  image: python:latest
  script:
  - pip install awscli
  - aws s3 cp ./public_html s3://$PROD_S3_BUCKET_NAME/ --recursive --acl public-read
  - aws cloudfront create-invalidation --distribution-id $PROD_CLOUDFRONT_DIST --paths "/*"
  only:
  - master
```

### Checking the results

If you now trigger a production deployment by pushing to the `master` branch, you'll see an extra step being logged once all the file copies have taken place:

![invalidation logs](/cloudfront/log.png "Invalidation log")

If you reload your live site after a minute or so, you should see your changes immediately viewable.