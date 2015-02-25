#Hacking APItools during 3scale internal hackathon

Last week, we had our first ever internal hackathon at 3scale. During two days we were able to take a break on our normal work life and start hacking on all the crazy ideas we had around our products (3scale, APItools or APIs.io).

As I've been playing a lot lately with APItools I surely wanted to build something on top of it. My great colleague Didier, wanted to hack the Hue lights from his flat. We've flet there was something to do there, connecting APItools and Hue lights.

Geeks and Engineers in general love analytics and dashboards. It's a great way for them to understand and visualize what's hapenning in the complex systems they are building. Visual feedback is really appreciated in a workspace, it informs you without disturbing you with emails, or pop-up notifications. It's a quiet notification.

And that's how we came up with our hack idea, we wanted to build a system to monitor the health of your APItools service. It could be useful if you are using APItools as a layer to distribute your own API or if your application is using APIs through APItools. And we would give a visual feedback using the Hue lights, with the basic light code of a stoplight, green, orange, red.

While we were drawing ideas on a whiteboard we realized that the full potential of this idea was it's modularity. Why only provide Hue light feedback when we can also send text, emails, play a song, or send data to an Arduino?

We would provide a solution for hackers to add directly from a nice UI, middleware to their APItools service. Kind of a "one-click deploy" middleware service.

In an other article we will give deeper explanations on how we have achieved this using APItools' API.

Today wanted to share our story pluging our Hue Lights into APItools.


##dd's story

### Intro
As it's been told before, enthusiastic geeks wanted to somehow reveal the status of an API using HUE lights. This was accomplished in the end with a bit of hacking skills, [great HUE API docs](http://www.developers.meethue.com/documentation/getting-started) and the powerful [APItools](https://www.apitools.com)

### HUE API, locally all run smoothly
Following the [HUE API docs](http://www.developers.meethue.com/documentation/getting-started), it's really stright forward to control your lights with their API within your local WiFi network, basically you follow these 3 steps:

 + Get your HUE bridge IP (you can retrieve this and more info about your bridge with their [broker service](https://www.meethue.com/api/nupnp) )
 + Authorize a new user
 + Start having fun controlling your lights with API calls! Yay!

However, we wanted to remote control our HUE bridge from our APItools Middleware, meaning not from our WiFi network but making calls from the vast Internet! And then things got a bit tricky...

### Controlling over Das Internet
To address your bridge over the internet, first you have to get an account from [Meet HUE](https://my.meethue.com), then you can use their web app and if you want to do it from your app, AFAIK, there's no way to automatically authenticate with your account and make calls to your bridge. So now it's when you set `hack_mode: true`.

#### A token, the holy token, that's what we need
We couldn't find a way to get it with an API call and at the time of this blog post, there's no documentation about it. But that didn't stop us, we continued with our hacking quest! What we had to do was:

1. From your browser `www.meethue.com/en-US/api/gettoken?devicename=<APP_NAME>&appid=hueapp&deviceid=<BRIDGE_ID>`
  * Where `APP_NAME` is whatever your hacking project, app, pet, etc name is and `BRIDGE_ID`, as mentioned before, you can get it from their [broker service](https://www.meethue.com/api/nupnp).
2. After accepting the device pairing, go to "Settings" and then "My apps". You will find a de-activate link for your HUE App, inspect the element and get the `tokenId` param.

#### Making tokenized calls
Now that we got the token, we can make calls to the API like this:

* URL: `https://www.meethue.com/api/sendmessage`
* Method: `POST` (always)
* URL Params:
    `token=<TOKEN>` (the holy one)
* Request headers
    `content-type=application/x-www-form-urlencoded`
* body
    `clipmessage={ bridgeId: "<BRIDGE_ID>", clipCommand: { url: "/api/0/<ENDPOINT>", method: "<METHOD>", body: <JSON_BODY> } }`
    * BRIDGE_ID is the same one you obtained earlier
    * ENDPOINT are the API endpoints listed in the official API, but we replaced the `username` by `0`
    * METHOD the same  methods as documented in the official API
    * JSON_BODY the command body, for example { "on": true }

When our request was successfully made, the response will look like this `{"code": 200, "message": "ok", "result": "ok"}`
Either way, it will display any eventual error:`{"code": 109, "message": "I don't know that token.", "result": "error"}`

### Shooting calls from a Middleware
Now that we are succesfully making calls to the API, we are ready to code our Middleware! As you might already know, [Middlewares are written in LUA](https://docs.apitools.com/docs/middleware/), and that's good! it's super easy following the [Middleware LUA API docs](https://rawgit.com/APItools/monitor/master/doc/index.html) and if you never coded LUA before, [learn it in 15 minutes](http://tylerneylon.com/a/learn-lua/).

Then you will be able to craft something like this:
```lua
  local hue_url = "https://www.meethue.com/api/sendmessage"
  local hue_token = "YoUrHoLymosTPReCIOUSToKENThatYOUProUDlYHaCKEdBeFORe"
  local hue_params = "?token=" .. hue_token
  local hue_bridge_id = "y0urbr1dg31d"
  local hue_status
  
  if error_codes[value] == "success" then
    hue_status =  25000 -- greenish
  elseif error_codes[value] == "warning" then
    hue_status =  10000 -- orange
  else
    hue_status =  0 -- red
  end
  
  body = json.encode({
          bridgeId= hue_bridge_id,
          clipCommand= {
            url= "/api/0/groups/0/action",
            method= "PUT",
            body= {
              on= true,
              hue= hue_status
            }
          }
        })
  
  hue_response = http.post(
                  hue_url .. hue_params,
                  "clipmessage=" .. body,
                  {
                    headers = {
                      ['Content-Type'] = 'application/x-www-form-urlencoded'
                    }
                  }
                )
  
  console.log("hue_response", hue_response)
```

Nice, isn't it!? So, this handcrafted code could be explain as it follows:

  1. We set our credentials and parameters to succesfully make use of our bridge via HUE API,
  2. then we set our `hue_status` variable with the HUE colour code we magically obtain from `error_code[value]` (this will be explain in a next blog post, but for now and in order to you be safe in the knowledge that you're not missing anything, there we store either "success", "warning" and "error" depending on the status code made in the last X minutes calls to the API we want to use.),
  3. the body we will send is constructed and..
  
  4. we finally made the call and log the response
  
### Wrapping up
As you can tell by now, we had a really fun time hacking HUE lights with APItools. This digest version of our story doesn't say in how many different ways we use APItools, like in debugging our calls to the API we wanted to monitor, the HUE API itself and then our Middleware calls to the HUE API, meaning debugging APItools with APItools (YEAH! INCEPTION!). Now this proof of concept is really valuable since the possibilities of using APItools with HUE lights are endless and still they could been associated with other APIs/Gadgets/etc.
