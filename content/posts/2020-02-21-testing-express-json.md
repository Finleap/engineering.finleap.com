---
title: "Unit-testing a JSON-serving API with Express"
date: 2020-02-21T13:39:29+01:00
draft: true
summary: Here's how you quickly nail up a basic unit test for an Express server acting as a  JSON-serving API.
author: Tim Duckett
author_email: tim.duckett@finleap.com
---
Here's a super-simple process for quickly nailing up a basic unit test of an Express server acting as a JSON-serving API. It uses a test against a JSON schema, which allows you to test the JSON payload without needing to delve into the values property-by-property. The trade-off is needing to generate a JSON schema in the first place, but hey - nothing in life is free, and there are tools available to remove some of the heavy lifting.

# Prerequisites

An Express app serving JSON in response to requests, e.g.

`GET http://myserver/endpoint` returns

```
{
    "personId": "8b50e33b9fe144b295e44f94f15dc856",
    "accountId": "604ceb8cabb84cca90a5336b5df53691",
    "accountBalance": 123456
}
```

# Install dependencies

As a minimum, you'll need:

* Chai
* Chai-HTTP
* Chai-JSON-Schema

Install with `yarn add <package> -D`

# Set up the test dependencies

Create a unit test file, import all the dependencies, and set up Chai:

{{< highlight js >}}
const chai = require('chai');
const chaiHttp = require('chai-http');
const chaiJsonSchema = require('chai-json-schema');
const app = require('<path_to_your>/app');

const { expect } = chai;

chai.use(chaiHttp);
chai.use(chaiJsonSchema);
{{< / highlight >}}

# Create a schema file

We'll use `JSON Schema` to validate the JSON payload that the server returns.  Create a `schema.js` file in the test directory and define it:

{{< highlight js >}}
const postSeedSchema = {

    "definitions": {},
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "title": "The Root Schema",
    "required": [
      "personId",
      "accountBalance"
    ],
    "properties": {
      "personId": {
        "$id": "#/properties/personId",
        "type": "string",
        "title": "The Personid Schema",
        "default": "",
        "examples": [
          "61df26122dd343448a8a7b8a3cadd905"
        ],
        "pattern": "^(.*)$"
      },
      "accountBalance": {
        "$id": "#/properties/accountBalance",
        "type": "integer",
        "title": "The Accountbalance Schema",
        "default": 0,
        "examples": [
          123456
        ]
      }
    }
  };

module.exports = ['postSeedSchema'];
{{< / highlight >}}

Life is too short to create JSON schemas by hand, so you can cheat and use a generated schema by pasting your JSON payload into the generator at [https://jsonschema.net/](https://jsonschema.net/).

# Writing the test

{{< highlight js >}}
describe('App', () => {

  describe('/endpoint', () => {

    /* Let's test if things work at all */

    it('responds with status 200', (done) => {
      chai.request(app)
        .get('/endpoint')
        .end((err, res) => {
          expect(res).to.have.status(200);
          done();
        });
    });

    /* Naive way of testing the returned JSON */

    it('responds with the correct JSON response', (done) => {
      chai.request(app)
        .get('/endpoint')
        .end((err, res) => {
          expect(res.body).to.have.property('personId');
          expect(res.body).to.have.property('accountId');
          expect(res.body).to.have.property('accountBalance');
          expect(res.body).to.have.property('accountAvailableBalance');
          expect(res.body).to.have.property('accountLimit');
          done();
        });
    });

    /* Now let's check that the JSON adheres to the schema */

    it('should have the correct JSON schema', (done) => {
    
      /* Import the schema */
      const schema = require('./schema');

      /* Test against the schema */
      chai.request(app)
      .get('/endpoint')
      .end( (err, res) => {
        expect(res.body).to.be.jsonSchema(schema);
        done();
      });
    });

  });

});
{{< / highlight >}}
