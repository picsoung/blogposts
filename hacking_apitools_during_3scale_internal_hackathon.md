#Hacking APItools during 3scale internal hackathon

Last week, we had our first ever internal hackathon at 3scale. During two days we were able to take a break on our normal work life and start hacking on all the crazy ideas we had around our products (3scale, APItools or APIs.io).

As I've been playing a lot lately with APItools I surely wanted to build something on top of it. My great colleague Didier, wanted to hack the Hue lights from his flat. We've flet there was something to do there, connecting APItools and Hue lights.

Geeks and Engineers in general love analytics and dashboards. It's a great way for them to understand and visualize what's hapenning in the complex systems they are building. Visual feedback is really appreciated in a workspace, it informs you without disturbing you with emails, or pop-up notifications. It's a quiet notification.

And that's how we came up with our hack idea, we wanted to build a system to monitor the health of your APItools service. It could be useful if you are using APItools as a layer to distribute your own API or if your application is using APIs through APItools. And we would give a visual feedback using the Hue lights, with the basic light code of a stoplight, green, orange, red.

While we were drawing ideas on a whiteboard we realized that the full potential of this idea was it's modularity. Why only provide Hue light feedback when we can also send text, emails, play a song, or send data to an Arduino?

We would provide a solution for hackers to add directly from a nice UI, middleware to their APItools service. Kind of a "one-click deploy" middleware service.

In an other article we will give deeper explanations on how we have achieved this using APItools' API.

Today wanted to share our story pluging our Hue Lights into APItools.


##DD's story
  
  
