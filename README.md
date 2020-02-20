![Live Hugo build & deploy](https://github.com/Finleap/finleap.tech/workflows/Live%20Hugo%20build%20&%20deploy/badge.svg)

# Process

## Background
The Finleap engineering blog is a flat HTML site served from Amazon S3 via a Cloudflare distribution. The site is built by the Hugo rendering engine, which is triggered by changes on the branches on the Gitlab service. Deployments to the S3 bucket are made automatically as part of the CI process.

Commits to branches other than master trigger a build and deployment to https://staging.engineering.finleap.com

Commits to the master branch trigger a build deployment to https://engineering.finleap.com

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

1. (Optional) Run the Hugo server `hugo server -D` (The `-D` switch forces building of draft posts)
1. Open `http://localhost:1313`
1. Create a new branch `git checkout -b yyyy-mm-dd/post-name` where `yyyy-mm-dd` = date (e.g. `2019-08-26`) and `post-name` = dash-separated post name (e.g. `my-new-post-about-kittens`)
1. Create a new post: `hugo new posts/yyyy-mm-dd-test-post.md`
1. Edit and save post - open `/posts/yyyy-mm-dd-test-post.md`, edit using Markdown syntax, and save
1. Update the post metadata:
    `draft`: false to publish post on live site (always visible on staging)
    `summary`: short summary of the post which is displayed on the homepage (2 - 3 lines)
    `author`: author name
    `author_email`: author email address
1.  Generate the new post: 	`hugo`
1.	Commit the changes to the local repo: `git add && git commit -am 'New post about kittens'`
1.	Push to the remote repo	`git push origin /yyyy-mm-dd/post-name`
1.	Check the post on the staging environment:	`https://staging.engineering.finleap.com`
1.	Create a merge request on Gitlab - click on the `Create Merge Request button` in the Gitlab UI

## Syntax highlighting

The bundled theme has some (semi-intelligent) syntax highlighting capabilities. 

To mark text as `monospaced`, enclose it in backticks.

```
To mark a block of text as monospaced like this, add three backticks above and below the text.
```

`{{< highlight js >}}`

To syntax-highlight a code snippet, wrap it in highlight markup.

`{{< / highlight >}}`

## Assets
Static assets (e.g. images) should be added in a sub-folder below the `./static` folder, and referenced in the markdown with  

```
![image alt text](/<subfolder>/<image name>#<optional anchor name)
```

To keep things tidy, please create a folder per post, naming it to relate to the branch/post, e.g.

branch name is `2020-02-20/my-new-post-about-kittens`

folder name should be `kittens`

# Live deployment

1. Checkout the merge candidate branch: `git mr origin <merge request number>` e.g. `git mr origin 2`
1. Check the proposed amendment in the `mr-origin-<MR number>` branch	
1. Switch to the master branch `git checkout master`
1. Merge the changes: `git merge --no-ff mr-origin-<MR number>`
1. Push the merged changes back to the repo: `git push origin master`
1. Check that the merge was successful and the changes have been deployed (nb updates will be available only once the Cloudfront distribution updates) `https://engineer.finleap.com`