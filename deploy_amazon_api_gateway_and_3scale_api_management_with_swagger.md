#Deploy Amazon API gateway and 3scale API management with Swagger

blabla swagger

schema

3scale

## On Amazon
### Create lambda functions
### Add stuff to your Swagger

In order to import your API into Amazon API gateway you will need to add AWS specific tags to your endpoints in your spec.

You would have to add those tags on each endpoint you want to expose in the gateway.

```
{
    "x-amazon-apigateway-auth": {
        "type": "none"
    },
    "x-amazon-apigateway-integration": {
        "type": "AWS",
        "uri": "arn: aws: apigateway: us-west-2: lambda: path/2015-03-31/functions/arn: aws: lambda: us-west-2: MY_ACCT_ID: function: helloWorld/invocations",
        "httpMethod": "METHOD",
        "requestTemplates": {
            "application/json": "{'body': $input.json('$'),'user_key': '$input.params('user_key')','resourcePath': '$context.resourcePath'}"
        },
        "requestParameters": {},
        "cacheNamespace": "cache-namespace",
        "cacheKeyParameters": [],
        "responses": {
            "-": {
                "statusCode": "200",
                "responseParameters": {},
                "responseTemplates": {
                    "application/json": ""
                }
            }
        }
    }
}
```

Customize this snippet with your own values. 

`MY_ACCT_ID` corresponds to your [amazon account id](http://docs.aws.amazon.com/general/latest/gr/acct-identifiers.html).
You can find it on your ARNs. For example on top of your lambda function. 

(image)

change `METHOD` to the corresponding HTTP method.
You can also modify the `requestTemplates` and the `responses` objects.
in `requestTemplates` the default `body, user_key` and `resourcePath` are mandatory to make the integration work.

### Import Swagger into Amazon API gateway

Once your Swagger spec is ready, you can import it using the [AWS swagger import tool](https://github.com/awslabs/aws-apigateway-swagger-importer).
Download it, assemble it, and deploy your Swagger.

If everything worked fine, you should have a new API available on your API gateway UI with all the endpoints and integrations you have defined.


## On 3scale

3scale let's you manage a developer community around your API.

To do the following steps you need first to create an account on [3scale](http://3scale.net).

### Import Swagger

Install the 3scale CLI tool

`npm install node-3scale-cli`

Setup two environements variables:

```
export THREESCALE_PROVIDER_KEY=<3scale_provider_key>
export THREESCALE_ID=<3scale_id>
```

The `THREESCALE_PROVIDER_KEY` could be found on your Account page in your 3scale dashboard.
(screenshot)

Your `3scale ID` corresponds to the URL you were given `THREESCALE_ID.3scale.net`.

Once the environement variables are setup you can run the following command:

`3scale-cli -f /path/to/swagger.json`

This will create a service service on 3scale, and special metric for each endpoint of your API. That will help to get relevant analytics.

### Use Developer portal 
