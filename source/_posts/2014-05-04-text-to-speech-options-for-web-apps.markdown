---
layout: post
title: "Text-To-Speech Options for Web Apps"
permalink: "text-to-speech-options-for-web-apps"
date: 2014-05-04 14:07:46 -0300
comments: true
categories: 
---

While I was building [PoemToday](http://www.poemtoday.com/), I wanted to add a feature that enabled users to play any poem aloud in a robot voice. If the inspiration for PoemToday, [The Writer's Almanac](http://writersalmanac.publicradio.org/), had Garrison Keillor’s voice coming over the NPR radio waves every morning, then my app would have the voice not of a killer robot, but a Keillorbot.

This proved to be easier said than done. First, there was a dearth of good quality options for text-to-speech that fit right into Rails. Second, with the Twilio implementation I chose to make, there were a few configuration steps between Twilio and Rails that were neither obvious nor well documented.

##Current TTS Options for Apps

Currently there’s a dearth of good quality text-to-speech options for web apps. I can’t really speak to mobile and desktop apps, but I believe those those platforms offer native TTS capabilities which developers can hook into. 

Some web app options, like AT&T, were paid services that seemed to actually require more of an integration hurdle and impose all kinds of usage restrictions. Others like the [tts](https://github.com/c2h2/tts) gem relied on a Google Translate hack which limited the spoken text to only 100 characters, about a sentence. A haiku might fit under that limit, but not a medium-length or longer poem. I also reached out to [Wit.ai](https://wit.ai/), but they said they don’t offer TTS yet. 

The better options were [Twilio](https://www.twilio.com/docs/howto/twilio-client-text-to-speech) and [tts-api.com](http://tts-api.com/). Twilio makes text-to-speech available through Twilio.Device, an API object that effectively turns your users’ browser into a phone or “soft device”. Meanwhile tts-api.com is just a one-page website with a single textarea on a form that returns an mp3 audio file when submitted. Nice and simple! 

Even though it was probably the best option in terms of performance, I didn’t want to download and store 6,000 mp3 files from tts-api.com, one for each poem, and rack up hosting fees. I could’ve also had the mp3 file from tts-api.com pop up in a new window, but I dislike pop ups (they’re terrible user experience) so I went with Twilio.

Also, while researching this post, I found the [espeak-ruby](https://github.com/dejan/espeak-ruby) gem, which I’ll haven’t tried out... I’ll have to come back soon and post an update. In the meantime, I’m glad I got a chance to figure out the Twilio implementation as I learned a lot. 

See the steps I took to implement [Twilio text-to-speech in Rails 4](/twilio-text-to-speech-in-rails-4).