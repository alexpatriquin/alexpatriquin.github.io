---
layout: post
title: "Twilio Text-To-Speech in Rails 4"
permalink: "/twilio-text-to-speech-in-rails-4"
date: 2014-05-04 15:06:41 -0400
comments: true
categories: [Tutorials, Rails 4, NLP]
---

This post walks through the steps I took to set up text-to-speech in a Rails 4 app. To learn about other options for implementing text-to-speech in web apps generally, see [Text-To-Speech Options for Web Apps](/text-to-speech-options-for-web-apps).

Also, the Twilio blog has a tutorial worth reading, [Integrating Twilio With Your Rails 4 App](https://www.twilio.com/blog/2014/02/twilio-on-rails-integrating-twilio-with-your-rails-4-app.html), but it does not cover setting up the crucial [Twilio.Device](https://www.twilio.com/docs/client/device) or TTS in Rails 4.

Before we get started, note that Twilio.Device only works when connected to the internet. So you can’t test Twilio.Device from localhost. Instead, you’ll need to expose your localhost to the internet with a utility called ngrok. (See #9 below.)

In order to set up Twilio TTS in a Rails 4 app, we need to complete 10 steps.

1. Sign up for Twilio
2. Install the twilio-ruby gem
3. Add endpoint for Twilio
4. Disable DSRF Detection on Twilio endpoint
5. Create a TwiML App
6. Setup Capability Tokens
7. Put Capability Token and text in an HTML form
8. Add Twilio.Device javascript
9. Setup ngrok to test
10. Deploy

<!-- more -->

For this walkthrough, I'll refer to code examples from [PoemToday](http://www.poemtoday.com/), a Rails 4 app that generates random poems from user input and reads them aloud. Let's begin.

###1) Sign up for Twilio

Easy enough. You’ll need an Account `SID` (short for “security identifier”) and `Auth Token`. Remember to hide these keys in `secrets.yml` if you’re pushing up to a public repo.

###2) Install the twilio-ruby gem

Twilio provides an [official ruby wrapper](https://github.com/twilio/twilio-ruby). We’ll use the DSL from the gem to create TWiML, or Twilio XML, and to create Capability Tokens.

###3) Add endpoints for Twilio

We’re going to create a POST endpoint for Twilio to tap into our app and receive the text we want read aloud. To do this in PoemToday, I added a `voice` action on my poems controller.

{% codeblock %} post 'poems/voice' => 'poems#voice' {% endcodeblock %}

Then, within the PoemsController, I setup the `voice` action to build a Twilio Response Object with the text we want spoken (sent as `params`). The method utilizes Twilio's [Say](https://www.twilio.com/docs/api/twiml/say) verb and renders the `response` as TwiML. 

```ruby Twilio Endpoint Example https://github.com/alexpatriquin/poem-today/blob/master/app/controllers/poems_controller.rb source
class PoemsController < ApplicationController
  after_filter :set_header, only: :voice
  include Webhookable

  def voice
    response = Twilio::TwiML::Response.new do |r|
      r.Say "#{params[:poem_content]}"
    end
    render_twiml response
  end
end
```

This code makes use of the `set_header` and `render_twiml` helper methods from the Twilio Rails 4 tutorial, which we can put into a `Webhookable` module in Concerns.

```ruby Webhookable Concern https://github.com/alexpatriquin/poem-today/blob/master/app/controllers/concerns/webhookable.rb source

module Webhookable
  extend ActiveSupport::Concern
 
  def set_header
    response.headers["Content-Type"] = "text/xml"
  end
 
  def render_twiml(response)
    render text: response.text
  end
end
```

One way to think about this code is that we’re creating an endpoint in a special variant of XML for Twilio to come and read. We’re creating a private API for Twilio.

###4) Disable CSRF Protection on Twilio endpoint

By default, Rails 4 blocks 3rd parties from `POST`ing, so as to prevent CSRF attacks. To accomplish this, Rails generates a random token when a form is created and then checks the token when the form is submitted. We want to accept a `POST` from Twilio, so we'll disable CSRF detection for the voice controller action.

```ruby
class PoemsController < ApplicationController
  skip_before_action :verify_authenticity_token, only: :voice

```

###5) Create a TwiML App

Now that we have a permitted endpoint, we can create a new TwiML App, which is just a set of URLs that tells Twilio what to do when it receives a call via telephone or Twilio.Device.

Create a new TwiML App under Dev Tools in your account and enter the `POST` endpoint (ie, `http://poemtoday/poems/voice`) as the `Voice Request URL`. After saving, note your new TwiML App’s `SID`, which we’ll need to create Capability Tokens.

###6) Setup Capability Tokens

In order to invoke Twilio.Device, users need to have a valid [Capability Token](https://www.twilio.com/docs/client/capability-tokens). Tokens are valid for 24 hours and it’s better for security reasons to give each of your users their own token. I actually create one on each poem page load.

Tokens can have Incoming or Outgoing Connection capabilities, or both. Since we want Twilio to `POST` to  our app, we’ll configure these Capability Tokens with an Outgoing Connection and pass in our TwiML App's `SID`, which lets Twilio know where to find our `voice` endpoint.

```ruby Generating Twilio Capability Tokens https://github.com/alexpatriquin/poem-today/blob/master/app/controllers/poems_controller.rb source
class PoemsController < ApplicationController

  def show
    @poem = Poem.find(params[:id])
    new_twilio_token
  end

  def new_twilio_token
    capability = Twilio::Util::Capability.new(ENV["TWILIO_ACCOUNT_SID"],ENV["TWILIO_AUTH_TOKEN"])
    capability.allow_client_outgoing(ENV["TWILIO_APPLICATION_SID"])
    @token = capability.generate
  end
end
```

Note that we’re creating `@token`, an instance variable on the controller, so that we can pass it up to the view.

###7) Put Capability Token and text in an HTML form

As we saw above in PoemsController, our Twilio Response Object accepts the text to be spoken as params, which we can send via form submission. We can use the HTML `data` attribute tag to pass the `@token` to javascript. 

[{% img center ./images/post_images/voice-icon.png 300 %}](http://www.poemtoday.com/poems/5706?keyword=voice)

PoemToday actually uses a hidden form and the `volume-up` icon from [Font Awesome](http://fortawesome.github.io/) for a submit button.

```erb
<%= hidden_field_tag "content", "#{@poem.content}", id: "poem-content" %>
<%= content_tag "div", id: "token", data: {token: @token } do %>
<%= button_tag fa_icon("volume-up lg"), id: "keillorbot" %>
```

###8) Add Twilio.Device javascript

Twilio’s TTS works through Twilio.Device, an API object. Twilio.Device serves as the main entry point for connecting with Twilio. For TTS, a connection can be understood as both a telephone and an api call. Twilio.Device connects, sends the relevant text to Twilio, holds open a port which receives an audio reading of the text, and then disconnects / hangs up.

Twilio.Device is only available on pages that have [twilio.js](https://www.twilio.com/docs/client/twilio-js). PoemToday is mostly poem pages, so I added a `javascript_include_tag` to my layout.

```javascript
<%= javascript_include_tag "//static.twilio.com/libs/twiliojs/1.1/twilio.min.js" %>
```

Finally, we’re ready to add the javascript that will invoke Twilio.Device with the user’s token and pass the text we want read aloud (ie, `#poem-content`). The code below disables the button while the connection is active.

```javascript Twilio.Device Example https://github.com/alexpatriquin/poem-today/blob/master/app/assets/javascripts/poems.js source
$(document).ready(function() {
  var token = $('#token').data('token');
  Twilio.Device.setup(token,{"debug":true});
  $("#keillorbot").click(function() {
      speak();
  });
  
  function speak() {
      var poem_content = $("#poem-content").val(); 
      $('#keillorbot').attr('disabled', 'disabled');
      Twilio.Device.connect({ 'poem_content':poem_content });
  }

  Twilio.Device.disconnect(function (conn) {
      $('#keillorbot').removeAttr('disabled');
  });
});
```

###<a name="ngrok"></a>9) Setup ngrok to test

As mentioned above, Twilio.Device only works when connected to the internet. To test, you’ll need to make localhost accessible via the internet with a utility such as ngrok. Twilio provides a good tutorial on [how to setup ngrok](https://www.twilio.com/blog/2013/10/test-your-webhooks-locally-with-ngrok.html).

Be sure to create a second TwiML App with your currently running ngrok address as your root domain. It should look something like ```http://3eb3c9ba.ngrok.com/poems/voice```.

###<a name="deploy"></a>10) Deploy

That’s it! If you’ve completed all the steps above, you should be good to go with adding text-to-speech to your Rails 4 app. 

If you run into any trouble or have any questions, feel free to ask me for help in the comments below or [message me on Twitter](https://twitter.com/apatriq).