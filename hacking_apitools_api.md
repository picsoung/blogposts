#Hacking APItools API

You may remember the last story we've shared about our internal hackathon at 3scale. Well, we didn't tell you everything about it, we have more to share with you :)

During our internal hackathon, [Didier](https://twitter.com/ddcsare 'Didier on Twitter') and myself decided to build an API health status light using APItools and [Hue light](https://www.apitools.com/apis/philips-hue 'Hue light'). The hue light will turn green if everything goes well, and red if there is an issue with the API response (4XX errors).

As we stated in [our previous story](https://docs.apitools.com/blog/2015/02/25/hacking-apitools-during-the-3scale-internal-hackathon.html 'Hacking APItools during the 3scale internal hackathon') we wanted to build to a `a "one-click deploy" middleware service.`
The idea was to authenticate users with their APItools account, they'd select a monitor and a service, and we'd add our middleware to that. As simple as one click for the user and a bit more complicated from the hacking side.

#### Using an undocumented API
APItools' own API is not really documented and not clearly exposed as an API. However, it is consumed by APItools itself. So we went back and forth doing stuff in the APItools UI and analized the XHR requests that were made to see what was needed. We also used this [blog post which contains some info about APItools' API](https://docs.apitools.com/blog/2014/08/21/traffic-monitor-api.html 'About APItools API'). 
And with the help of our APItools engineers [Michal](https://twitter.com/_mikz 'Michal on Twitter') and [Enrique](https://twitter.com/otikik 'Enrique on Twitter'), we were able to make it work.

When we started, this is the flow he had in mind:

1. Authenticating a user
2. Retrieving monitors
3. Retrieving services from monitor
4. Adding middleware to the service

#### Authenticating a user
At first we thought that people would just give us a username and a password and we would log easily them in as there is nothing like OAuth in APItools. But it turned out that authentication with username and password was not available in the API.

Instead we could auth the user with their APItools API key and the monitor id they wanted to use. However, we wouldn't be able to list existing monitors for a user. That reduced our idea but it was also simpler to hack during a hackathon.

Your API key can be found under the settings menu:

**Screenshot**

We also needed the monitor id which can be found by clicking on the monitor link. The URL should look like this `http://MONITOR_ID.my.apitools.com`

Once you have both the API key and the monitor Id you can authenticate the request to APItools.

Make a simple GET request to `https://API_KEY@MONITOR_ID.my.apitools.com/`
In the response of this request extract the cookie `XSRF-TOKEN` and save it for later.

#### Play with services
Once you are authenticated on one monitor you can play with the services attached to that monitor.

For example, if you want to get all the services of a monitor you can call `https://API_KEY@MONITOR_ID.my.apitools.com/api/services`
This will give you something like

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

In our case, during the hackathon we decided to keep it simple and just push a new service every time.
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

And a few placeholders that have to be taken care of:

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
If the response returns a 201 status it means that service was sucessfuly created. Store `data._id`, we will need it later.

####Adding middleware to the service
Once the new service has been created, middleware can be added. Middleware modules are small snippets of Lua code that are executed when you make a call through APItools.

Middleware modules can be turned on or off, depending on if you want them active or not. You can also change the order of the middleware modules choosing the order in which they are executed.

In theory using the API you can load all the existing middleware modules of a service and see in which order they are executed. But in our case we didn't need that and we simply flushed the existing middleware pipeline and pushed our own.

We have written a few snippets of Lua code we wanted to push as middleware modules.

The first thing to do is creating a middleware object.

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
As you can see middleware modules are identified by a unique Id, we created a function `uuid()` to generate one but you are free to use any id you want. 

The code in the middleware module has to be loaded from the file you already wrote. Feel free to change the name and the description.

Once you the middleware object is created you can push it to the service.


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

If everything went well you should have something like this

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

That's just an overview of how we used APItools' API in our hack. You could do much more without the time constraint of a hackathon. Imagine being able to push your middleware modules, dedicated to your API or to your users, in just few clicks.

You can find some hints for other API methods looking at the code on [Github](https://github.com/APItools/monitor/blob/master/lua/apps/api.lua).

Can't wait to see what you are going to build with it :)

Happy hacking!
