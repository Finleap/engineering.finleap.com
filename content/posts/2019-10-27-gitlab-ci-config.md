---
title: "Configuring GitLab CI"
date: 2019-10-27T17:43:01+01:00
draft: false
summary: "How to configure a GitLab CI YAML file"
author: Tim Duckett
author_position: Head of Engineering
author_email: tim.duckett@finleap.com
---
# The config file

GitLab continuous integration is controlled by the `.gitlab-ci.yml` file, which should exist in the root of the project's file structure. When the presence of this file is detected in response to a commit, the GitLab CI runner will kick off the relevant stages as defined in the `yml` file.

# Structure

The file has three main sections:

+ **variables** which can be used to substitute values in any of the stages
+ a **stages section** which defines which stages are present
+ **configuration for each job** defining in detail how the process should run.

# Variables

These should be listed underneath the `variables` key, e.g:

```
variables:
  PROD_S3_BUCKET_NAME: "myblog.cc"
  STAGING_S3_BUCKET_NAME: "staging.myblog.cc"
  PROD_BLOG_URL: "https://myblog.cc/"
  STAGING_BLOG_URL: "https://staging.myblog.cc/"
```

# Stages

The `stages` key lists all the stages that will run as part of a job. They run sequentially as listed, but where there are multiple jobs in a stage, these will run in parallel.

```
stages:
  - stage_build
  - prod_build
  - stage_deploy
  - prod_deploy
```

Here there are four stages, broken into two types - `stage`, which targets the staging environment and `prod` which targets the live environment.

# Job definitions

This is an example of a job definition:

```
build_production:
  stage: prod_build
  image: jguyomard/hugo-builder:latest
  script:
  - git submodule update --init --recursive
  - hugo --destination public_html --baseURL "${PROD_BLOG_URL}"
  cache:
    paths:
    - public_html
  artifacts:
    paths:
    - public_html
  only:
  - master
```

Breaking this down line by line:

**`build_production:`**

this is the name of the job, which should be unique

**`stage: prod_build`**

the name of the stage to which this job belongs. Where a stage has more than one job, all jobs will run in parallel


**`image: jguyomard/hugo-builder:latest`**

The Docker image that will be built and used to run this job - in this case, it's a minimum viable Hugo installation used to build the static site assets.

**`script:`**<br/>
**`- git submodule update --init --recursive`**<br/>
**`- hugo --destination public_html --baseURL "${PROD_BLOG_URL}"`**

The Bash script(s) that will run once the Docker image has been spun up. Variables can be substituted using the `${VARIABLE_NAME}` syntax.

**`cache:`**<br/>
&nbsp;&nbsp;**`paths:`**<br/>
&nbsp;&nbsp;**`- public_html`**

The paths that should be cached internally during the job. Caches don't get persisted outside of the lifespan of the job itself.

**`artifacts:`**<br/>
&nbsp;&nbsp;**`paths:`**<br/>
&nbsp;&nbsp;**`- public_html`**

Paths that _should_ be persisted between jobs, so can be accessed by other jobs after this one has completed. In this case, Hugo will write the site assets to the `public_html` directory, and these files will be transferred to S3 in the next deployment stage.

**`only:`**<br/>
**`- master`**

Git branches where commits will trigger this job - any commit to the `master` branch will trigger this job.

The opposite of `only:` is `except:`, so

**`except:`**<br/>
**`- staging`**

will run the job on all branches _except_ `staging`.

# The full sample `.gitlab-ci.yml`

```
variables:
  PROD_S3_BUCKET_NAME: "trashpanda.cc"
  STAGING_S3_BUCKET_NAME: "staging.trashpanda.cc"
  PROD_BLOG_URL: "https://trashpanda.cc/"
  STAGING_BLOG_URL: "https://staging.trashpanda.cc/"

stages:
  - stage_build
  - prod_build
  - stage_deploy
  - prod_deploy

prod_build:
  stage: prod_build
  image: jguyomard/hugo-builder:latest
  script:
  - git submodule update --init --recursive
  - hugo --destination public_html --baseURL "${PROD_BLOG_URL}"
  cache:
    paths:
    - public_html
  artifacts:
    paths:
    - public_html
  only:
  - master

stage_build:
  stage: stage_build
  image: jguyomard/hugo-builder:latest
  script:
  - git submodule update --init --recursive
  - hugo --buildDrafts --destination staging_html --baseURL "${STAGING_BLOG_URL}" --environment staging
  cache:
    paths:
    - staging_html
  artifacts:
    paths:
    - staging_html
  except:
  - master

prod_deploy:
  stage: prod_deploy
  image: python:latest
  script:
  - pip install awscli
  - aws s3 cp ./public_html s3://$PROD_S3_BUCKET_NAME/ --recursive --acl public-read
  only:
  - master

stage_deploy:
  stage: stage_deploy
  image: python:latest
  script:
  - pip install awscli
  - aws s3 cp ./staging_html s3://$STAGING_S3_BUCKET_NAME/ --recursive --acl public-read
  except:
  - master
```
