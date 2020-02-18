# Process

## Background
The Finleap engineering blog is a flat HTML site served from Amazon S3 via a Cloudflare distribution. The site is built by the Hugo rendering engine, which is triggered by changes on the branches on the Gitlab service. Deployments to the S3 bucket are made automatically as part of the CI process.

Commits to branches other than master trigger a build and deployment to https://staging.finleap.tech

Commits to the master branch trigger a build deployment to https://finleap.tech

Pages are written in Markdown.

# Staging authentication
The staging site is protected by HTTP Basic Auth, controlled by a Lambda function which is invoked by the CloudFront distribution for each request.

The Lambda function `arn:aws:lambda:us-east-1:804609905630:function:StagingSiteAuthenticator` is located in the `US-EAST-1` region

Username and password are defined in the Lambda function itself:

Username: `staging`
Password: `finleap2020`

# Prerequisites
* (Optional) Local installed version of Homebrew
* (Optional) Local installed version of Go → https://gohugo.io/getting-started/installing/
* Local installed version of Hugo → https://gohugo.io/getting-started/installing/
* Create a local clone of the project repo `git@gitlab.com:finleap/finleap-tech-blog.git`
* (Optional) for quick checkout of Gitlab merge commits, add the following line to `~/.gitconfig`
** `mr = !sh -c 'git fetch $1 merge-requests/$2/head:mr-$1-$2 && git checkout mr-$1-$2' -`

# Creating a new post

1. (Optional) Run the Hugo server `hugo server -D`
1. Open `http://localhost:1313`
1. Create a new branch `git checkout -b yyyy-mm-dd/post-name` where `yyyy-mm-dd` = date (e.g. `2019-08-26`) and `post-name` = dash-separated post name (e.g. `my-new-post-about-kittens`)
1. Create a new post: `hugo new posts/yyyy-mm-dd-test-post.md`
1. Edit and save post - open `/posts/yyyy-mm-dd-test-post.md`, edit using Markdown syntax, and save
1. Update the post metadata:
    `draft`: false to publish post on live site (always visible on staging)
    `summary`: short summary of the post which is displayed on the homepage
    `author`: author name
    `author_email`: author email address
1.  Generate the new post: 	`hugo`
1.	Commit the changes to the local repo: `git add && git commit -am 'New post about kittens'`
1.	Push to the remote repo	`git push origin /yyyy-mm-dd/post-name`
1.	Check the post on the staging environment:	`https://staging.finleap.com`
1.	Create a merge request on Gitlab - click on the `Create Merge Request button` in the Gitlab UI

## Assets
Static assets (e.g. images) should be added in a sub-folder below the `./static` folder, and referenced in the markdown with  

```
![image alt text](/<subfolder>/<image name>#<optional anchor name)
```

# Live deployment

1. Checkout the merge candidate branch: `git mr origin <merge request number>` e.g. `git mr origin 2`
1. Check the proposed amendment in the `mr-origin-<MR number>` branch	
1. Switch to the master branch `git checkout master`
1. Merge the changes: `git merge --no-ff mr-origin-<MR number>`
1. Push the merged changes back to the repo: `git push origin master`
1. Check that the merge was successful and the changes have been deployed (nb updates will be available only once the Cloudfront distribution updates) `https://finleap.tech`