#Hacking APItools API

You may remember the last story we've shared about our internal hackathon at 3scale. Well we did not say tell you everything about it, we have more to share with you :)

During our internal hackathon, Didier and myself decided to build an API health status light using APItools and Hue light. The hue light will turn Green if everything goes well, and red if there is an issue on the API (4XX errors).

As we stated in our previous story we wanted to build to a `a "one-click deploy" middleware service.`
We would authenticate users on their APItools account, they select a monitor and a service, and we will add our middleware to it. Simple as one click for the user a bit more complicated from the hacking side.


## Using an undocumented API
APItools own API is not really documented and not clearly exposed as an API. But the tools that you are using everyday is consuming this API. So we went back and forth doing stuff on APItools UI and analysing the XHR requests that were made to see what we would need. 
With the help of our APItools engineers Michal and Enrique, we were able to make it working.

The flow we had in mind at the beginning of the hackathon was:

1. Authentication user
2. Retrieve monitors
3. Retrieve services from monitor
4. Add middleware to service

## Authenticating a user
At first we thought that people will just give us username and password and we will log them easily as there is nothing like OAuth in APItools. But it turns out that authentication with username and password was not available in the API.

Instead we can auth user with their APItools API key and the monitor id they want to use. So we won't have either the possibility to list existing monitors for a user. That reduced our idea but it was simpler to hack during a hackathon.

Your API key could be found under the settings menu

**Screenshot**

We will also need the monitor id, to get it just click on the name of your monitor, you will get to the dashboard for the corresponding monitor, the URL should be `http://MONITOR_ID.my.apitools.com`

Once you get both API key and monitor Id we can authenticate the request to APItools.

Make a simple GET request to `https://API_KEY@MONITOR_ID.my.apitools.com/`
In the response of this request extract the cookie `XSRF-TOKEN` and keep it for later.

## Play with services
Once you get authenticated on one monitor you can play with the services related to it.

For example if you want to get all the services of a monitor you can call `https://API_KEY@MONITOR_ID.my.apitools.com/api/services`
This will give you something similar to

```
[
    {
        "_created_at": 1401357490.031,
        "_updated_at": 1415824067.348,
        "endpoints": [
            {
                "code": "b83c5a5e",
                "url": "http://wservice.viabicing.cat/getstations.php?v=1"
            }
        ],
        "name": "Bicing",
        "_id": 4
    },
    {
        "_created_at": 1407826870.911,
        "_updated_at": 1417456661.228,
        "_id": 8,
        "endpoints": [
            {
                "url": "https://echo-api.herokuapp.com/",
                "code": "echo560bfa1c"
            }
        ],
        "name": "Echo API",
        "demo": "echo",
        "description": "Echo is simple service which responds for every request with JSON containing the request information. Like looking in the mirror. Useful for debugging middlewares."
    },
    {
        "_created_at": 1412951138.981,
        "_updated_at": 1428397938.496,
        "endpoints": [
            {
                "code": "proxy",
                "url": "*"
            }
        ],
        "name": "Proxy",
        "_id": 9
    }
]
```

in our case, during the hackathon we decide to go the simplest way and just push a new service every time.
Here is the call to create a new service on APItools

```
host = https://API_KEY@MONITOR_ID.my.apitools.com/api/services;
HTTP.post(host,{
    auth: API_KEY+":"+MONITOR_ID,
    headers: {"X-XSRF-TOKEN": XSRF_TOKEN, "Content-Type" : "application/json"},
    data: {
      endpoints:[{
        url: ENDPOINT_OF_THE_API,
        code: URLify2(NAME_OF_THE_SERVICE)
      }],
      name:NAME_OF_THE_SERVICE
    }
});
```

Couple of placeholders you have to take care of :

* API_KEY - APIKEY of APItools
* MONITOR_ID - Monitor ID where  you want to add the service
* XSRF_TOKEN - Token you got from previous call
* ENDPOINT_OF_THE_API - What is the endpoint URL of the API? ex: api.github.com
* NAME_OF_THE_SERVICE - Name of the service, could be anything, we use [URLify](https://www.npmjs.com/package/urlify) to escape special chars.

This will give you:

```
{
    statusCode: 201,
    content: '{
        "_created_at": 1431097935.906,
        "_updated_at": 1431097935.906,
        "endpoints": [
            {
                "url": "ENDPOINT_OF_THE_API",
                "code": "NAME_OF_THE_SERVICE"
            }
        ],
        "name": "NAME_OF_THE_SERVICE",
        "_id": 69
    }',
    headers: {
        'content-type': 'application/json',
        date: 'Fri, 08May201515: 12: 15GMT',
        server: 'openresty/1.5.11.1',
        'set-cookie': [
            'apitools_conrad_auth=OyhITbhzJvy876/duhushidhihsiuhiuhihih+1R9o3utntTI6Q4++PSMrQI/myYzfuWQ5b2HKRjEi9H72Ugb0+fHsi5RjYQ+;Path=/;Secure;HttpOnly;'
        ],
        'content-length': '151',
        connection: 'keep-alive'
    },
    data: {
        _created_at: 1431097935.906,
        _updated_at: 1431097935.906,
        endpoints: [
            [
                Object
            ]
        ],
        name: 'NAME_OF_THE_SERVICE',
        _id: 69
    }
}
```
If the response has a status 201 it means that service was sucessfuly created. Store `data._id` we wil need it for later.

##Push middleware
Now that we have created a new service we want to add middleware. Middleware are small snippet of Lua code that are executed when you make a call through APItools.
Middleware could be turn on or off, depending if you want them active or not. You can also change the order of middleware, so one will be executed before others.

In theory using the API you can load all the existing middleware of a service and see in which order they are executed. But in our case we wanted to do some simple for the user, so he did not have to think about it. We would simply flush the existing middleware pipeline and push our own.

We had write snippets of lua code we want to push as middleware.

First we have to create a middleware object.

```
mw_uuid = uuid(); //Generate a unique id to identify the middleware
code = // Load from lua file
middlewares = {};
middlewares[mw_uuid] = {
  active: true,
  code: code,
  description: "Middleware autogenerated by API Health Bar",
  name: "AutoDegeneratedMw",
  position: 0,
  uuid: mw_uuid
};
```
As you see a middleware is identified by a unique Id, we created a function `uuid()` to generate one but you are free to use any id you want. 
The code of the middleware has to be loaded from a file you already wrote.
Feel free to change the name and the description of the middleware.

Once you the middleware object created, you can push it to the service.


```
host = https://API_KEY@MONITOR_ID.my.apitools.com/api/services/serviceID/pipeline;
HTTP.post(host,{
  auth: API_KEY+":"+MONITOR_ID,
  headers: {"X-XSRF-TOKEN": XSRF_TOKEN, "Content-Type" : "application/json"},
  data: {
    _id: 1,
    middlewares: middlewares,
    service_id: SERVICE_ID,
  }
});
```

if everything went well you should have something like this

```
{
    statusCode: 200,
    content: '{
        "_created_at": 1431099220.474,
        "_updated_at": 1431099223.366,
        "middlewares": {
            "MIDDLEWARE_UUID": {
                "active": true,
                "position": 0,
                "code": YOUR_MIDDLEWARE_CODE,
                "name": "AutoDegeneratedMw",
                "uuid": MIDDLEWARE_UUID,
                "description": "Middleware autogenerated by API Health Bar"
            }
        },
        "_id": 71,
        "service_id": 71
    }',
    headers: {
        'content-type': 'application/json',
        date: 'Fri,08May201515: 33: 43GMT',
        server: 'openresty/1.5.11.1',
        'set-cookie': [
            'apitools_conrad_auth=NxNk0l4KmcS4h4jW6WCy4f+1R9o3utntTI6Q4++PSMrQI/myYzfuWQ5b2HKRhyMaIxBDryo4ekt76RdCMo;Path=/;Secure;HttpOnly;'
        ],
        'content-length': '2970',
        connection: 'keep-alive'
    },
    data: {
        _created_at: 1431099220.474,
        _updated_at: 1431099223.366,
        middlewares: {
            'MIDDLEWARE_UUID': [
                Object
            ]
        },
        _id: 71,
        service_id: 71
    }
}
```

That's just an overview of what we've used from APItools' API in our hack. You can do much more if you are not constraint by time. Imagine being able to push your middleware dedicated to your API to your users, in few clicks.

You can find some hints for other API methods looking at the code on [Github](https://github.com/APItools/monitor/blob/master/lua/apps/api.lua).

Can't wait to see what you are going to build with it :)

Happy hacking!


