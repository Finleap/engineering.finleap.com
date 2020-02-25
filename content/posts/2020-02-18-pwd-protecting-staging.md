---
title: "Password-protecting websites served from S3"
date: 2020-02-18T10:58:32+01:00
draft: false
summary: How to password-protect a site being served from S3 via a CloudFront distribution
author: Tim Duckett
author_position: Head of Engineering
author_email: tim.duckett@finleap.com
---

This [earlier post](https://finleap.tech/posts/web-serving-on-aws/) worked through the process of automating deployment of a Hugo blog to an Amazon AWS environment. Part of that included creating a staging instance which would deploy automatically in response to pushes to non-master branches in the Git repo.

While it's useful to be able to see work in progress content, making that available to whoever stumbles across the site isn't ideal - it would be better if the staging site was secured behind a login.

For most purposes, HTTP Basic Auth is enough - but because serving web content from S3 doesn't have any server-side processing capability, we need to find another way to password-protect the site.

Adding a Lambda function that checks for a username/password combination is a relatively straight-forward process, and balances security and complexity.

It's a multistage process:

# The process

- [The process](#the-process)
- [1. <a name="iam_role"></a>Creating a new IAM role](#1-creating-a-new-iam-role)
- [2. <a name="iam_permissions"></a>Updating the IAM role permissions](#2-updating-the-iam-role-permissions)
- [3. <a name="lambda_function"></a>Creating the Lambda function](#3-creating-the-lambda-function)
- [4. <a name="connecting-lambda"></a>Connecting the Lambda function to the CloudFront distribution](#4-connecting-the-lambda-function-to-the-cloudfront-distribution)

Make sure that you perform all steps while in the US-East-1 (N Virginia) region, as CloudFront can only access Lambda function in this region.

# 1. <a name="iam_role"></a>Creating a new IAM role

Assuming you're logged into the AWS console, switch to IAM, and select the *Roles* option from the left-hand sidebar. Then click the `Create Role` button, highlight the `AWS Service` block at the top of the page, and then select `Lambda` from the list of service. Now click `Next`.

In the `Filter Policies` box at the top, type `AWSLambdaExecute` to filter the available policies, and check the box next to the policy that's displayed. Then click `Next`.

![filter policies](/passwords/filterPolicies.png "Filter policies")

Skip the `Tags` screen by clicking on the `Review` button, then give the role a name - something like `S3StagingAuthenticator`, although it's only a hint to enable you to find the role in the next stage. Finally, click the `Create role` button to create the role and be returned to the list.

# 2. <a name="iam_permissions"></a>Updating the IAM role permissions

In the list of roles, click on the one that you've just created (you may need to the first few characters of the role's name into the filter box to narrow the list down.)

In the Summary page, click on the `Trust Relationships` tag, then click the `Edit trust relationship` button.
This will open up an editing window where you can amend the JSON document which defines the trust relationship for this role.

![edit trust](/passwords/editTrust.png "Edit trust")

Update the JSON to the following:

{{< highlight json >}}
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "lambda.amazonaws.com",
          "edgelambda.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
{{< / highlight >}}

Then click `Update Trust Policy` - this adds the `edgeLambda` service to the policy, allowing the Lambda function to be triggered by the CloudFront distribution.


# 3. <a name="lambda_function"></a>Creating the Lambda function

Now it's time to create the Lambda function which does the heavy lifting of checking the credentials provided.

Open the Lambda dashboard and click the `Create function` button. Leave the `Author from scratch` radio button selected, give the function a name - something like `stagingSiteAuthenticator` and select the `Node.js 10.x` runtime. Expand the `Execution role` section, select the `Use an existing role` radio button and select the role that you created in step 3 from the list. Now click the `Create function` button.

You'll now see the `Designer` screen.

![function code](/passwords/functionCode.png "Function code")

 In the `Function code` section, paste the following code into the editor window:

{{< highlight js >}}

exports.handler = (event, context, callback) => {
  
  checkCredentials(event, context, callback);
  
}

function checkCredentials(event, context, callback) {
  
  // Get the request and its headers
  const request = event.Records[0].cf.request;
  const headers = request.headers;

  // Specify the username and password to be used
  const user = '<your user name>';
  const pw = '<your password>';

  // Build a Basic Authentication string
  const authString = 'Basic ' + new Buffer(user + ':' + pw).toString('base64');

  // Challenge for auth if auth credentials are absent or incorrect
  if (typeof headers.authorization == 'undefined' || headers.authorization[0].value != authString) {
    const response = {
      status: '401',
      statusDescription: 'Unauthorized',
      body: 'Unauthorized',
      headers: {
        'www-authenticate': [{key: 'WWW-Authenticate', value:'Basic'}]
      },
    };
    callback(null, response);
  }

  // User has authenticated
  callback(null, request);
  
};
{{< / highlight >}}

Briefly, this code does the following:

* Defines a `checkCredentials` function that receives the HTTP headers in the `event` object
* Builds a basic auth challenge
* Checks the user's provided username and password values against the hard-coded values in the function
* Returns a `401 unauthorized` HTTP code if they don't match
* Passes the request if the match is made

Click the `Save` button in the top right to publish the function; then copy the `ARN` that's shown above the button.

Drop down the `Actions` button, and select the `Publish new version` option - give the version a description, and click the `Publish` button. 

![publish version](/passwords/publishVersion.png "publish version")

Then drop down the `Qualifiers` menu, and make a note of the highest number. This will increase by one every time you publish a new version of the function.

![qualifiers](/passwords/qualifiers.png "qualifiers")

# 4. <a name="connecting-lambda"></a>Connecting the Lambda function to the CloudFront distribution

With the Lambda function created, it's time to link it to the CloudFront distribution so that it's triggered when the initial request is made.

In the CloudFront administration panel, click on the `ID` of the distribution that corresponds to the staging site. This will open up the edit screen for this distribution - select the `Behaviors` tab.

Select the checkbox next to the `Default` path pattern and click the `Edit` button. This will open the `Default Cache Behavior Settings` page.  Towards the bottom there's the `Lambda Function Associations` which will currently have one `Origin Request` setting. Drop down the `Select Event Type` menu, and select the `Viewer Request` type. Then paste in the `ARN` that you copied in step 3, and edit it to add the version number of the Lambda function after a colon - so:

```
arn:aws:lambda:us-east-1:nnnnnnnnnnnn:function:StagingSiteAuthenticator:X
```
where `X` is the version number that you noted in step 3.

![function association](/passwords/functionAssociation.png "function association")

Click the `Yes, edit` button to save these changes, then click on the `Cloudfront Distributions` breadcrumb at the dop of the screen to return to the list of distributions.

The distribution that you've just edited will now have a status of `In Progress` - it can take up to 10 minutes for the changes to be pushed out into the content distribution network.  

![status](/passwords/status.png "status")

When the status changes to `Deployed`, hit the staging URL of your site, and you should now be presented with an authentication prompt:

Enter an invalid username and password pair, and you'll see an `Unauthorized` error. Enter the correct credentials, and the site will be loaded via the CloudFront distribution as before.

