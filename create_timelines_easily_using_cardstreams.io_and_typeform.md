#How To Create Timelines Ysing Cardstream and Typeform APIs

While I am preparing for StartupBus I got the chance to try out all the APIs we (3scale) decided to bring with us on the bus and expose to *buspreneurs*. Amongst the few selected APIs we have two of our awesome customers: TypeForm and Cardstream.

You may have heard about Typeform, they have a great self-service form creation tool. In a few clicks, you can build beautiful forms and share them with your users. And you know what? They just launched an API! Using the Typeform API you can create surveys and receive answers on webhooks. They just opened their beta and are making improvements to the API daily.

Cardstream just launched their new API too. Creating feeds or activity streams has never been easier.

In my example I imagined that I wanted to display user stories on my site. Users would submit their stories using Typeform and we woud generate the stream using Cardstream. Easy.

Here is a schema of the setup.

![Screenshot](https://dl.dropboxusercontent.com/u/2996126/site/cardstream_typeform_diagram.png)

The form is available [here](https://forms.typeform.io/to/gbuBwDsG58ectQ)

The final result can be found [here](https://dl.dropboxusercontent.com/u/2996126/site/cardstream.html):

##What you need
1. Typeform.io developer account
2. Carstreams.io developer account
3. APItools account

## APItools configuration
Create a first service pointing to the echo-api. We will use this service to receive the webhook sent by Typeform every time a user submits an answer.

Create a second service pointing to Cardstream API `https://api.cardstreams.io/v1`

##Typeform.io configuration

###Create the form
We are going to create a simple form with three text fields: one to get the name of the user, another one to get the title of the story and a last one to get the content of the story.

With just a simple call to the Typeform API we will generate this form and add a webhook. The answer of the call will give us useful information and we'll see how to use it in this tutorial.

Make a `POST` request on `https://api.typeform.io/v0.2/forms`
passing headers `X-API-TOKEN` with your Typeform token.
and the JSON content as follows:

```
{
  "title": "My first typeform",
  "webhook_submit_url":"https://URL_TO_APITOOLS_SERVICE_FOR_WEBHOOK/",
  "fields": [
    {
      "type": "short_text",
      "question": "What is your name?"
    },
    {
      "type": "short_text",
      "question": "Title of your post"
    },
    {
      "type": "long_text",
      "question": "What is the story of your life?",
      "description": "Please describe it within 50 characters",
      "required": true
    }
  ]
}
```
In this you will have to replace `URL_TO_APITOOLS_SERVICE_FOR_WEBHOOK` by the URL of the first service you created.

Once this call is executed you will receive an answer from the Typeform API. It should look like this:

```
{
    "id": "Dbu0eA2rX7qVlQ",
    "design_id": 1,
    "fields": [
        {
            "type": "short_text",
            "question": "What is your name?",
            "position": 0,
            "choices": null,
            "id": 207973
        },
        {
            "type": "short_text",
            "question": "Title of your post",
            "position": 1,
            "choices": null,
            "id": 207974
        },
        {
            "type": "long_text",
            "question": "What is the story of your life?",
            "position": 2,
            "choices": null,
            "id": 207975
        }
    ],
    "links": {
        "form_render": {
            "get": "https://forms.typeform.io/to/Dbu0eA2rX7qVlQ"
        }
    },
    "title": "My first typeform",
    "webhook_submit_url": "URL_TO_APITOOLS_SERVICE_FOR_WEBHOOK"
}
```

Keep this answer somewhere, you will need the URL of the form to test it out, as well as the id of each field.

## Cardstream configuration
On [Cardstream developer portal](https://developer.cardstreams.io/cardbuilder) you can create a new stream. Create one and save the ID somewhere.

## Add middleware to handle webhook
On the first service you have create we will now add a middleware module to pass the data to Cardstream.

Here is the code of the middleware:

``` 
return function(request, next_middleware)
  local response = next_middleware()
  local answers = json.decode(request.body).answers
  local title =""
  local user = ""
  local text = ""
  for i=1,#answers do
  	if(answers[i].field_id == ID_OF_FIELD_FOR_USERNAME) then
      user = answers[i].data.value
    end
    if(answers[i].field_id == ID_OF_FIELD_FOR_TITLE) then
      title = answers[i].data.value
    end
    if(answers[i].field_id == ID_OF_FIELD_FOR_STORY) then
      text = answers[i].data.value
    end
  end
  
  -- call to cardstream
  local streamID = "CARDSTREAM_STREAM_ID"
  local cardstreamURL = "https://CARDSTREAM_APITOOLS_URL/streams/"..streamID.."/cards"
  local r = http.json.post(cardstreamURL,{title=title.." by "..user,description=text})
  console.log(r)
  return response
end
```

In this code you will change:

`ID_OF_FIELD_FOR_USERNAME`, `ID_OF_FIELD_FOR_TITLE` and `ID_OF_FIELD_FOR_STORY` with the IDs given by Typeform.

`CARDSTREAM_STREAM_ID` ID of stream you have created on Cardstream

and `CARDSTREAM_APITOOLS_URL` URL of the APItools service linked to Cardstream.

## Test the flow
To test that everything is working, submit an answer on your form. You should see the webhook call going through the APItools service and then calling the Cardstream service.

At the end, in the Cardstream portal (preview tab), you should see the working result.

You can then generate the embed code and paste it on your site.

And voila, it was that easy :) There are endless integrations you can build using Typeform API or Cardstream, we'd love to hear about what you're working on!

If you are interested in participating in StartupBus you still have time to apply [here](https://northamerica.startupbus.com/apply/) and attend the [kickoff party](https://www.facebook.com/events/1587243618219465/) on June 3rd in San Francisco.	
