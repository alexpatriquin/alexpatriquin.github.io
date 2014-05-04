---
layout: post
title: "How I Built PoemToday"
permalink: "how-i-built-poemtoday/"
date: 2014-04-28 18:49:57 -0400
comments: true
categories: 
---

I graduated from the [Flatiron School](http://flatironschool.com/) last week and, after an exhilarating three months, had the chance to present my final projects at our Science Fair. Along with [Tender](http://tendermessenger.com/), I presented PoemToday, a random poem generator that's powered by user behavior. Here is the story of how it works and how I made it.

[{% img center ../images/post_images/stats-poem.png 500 %}](http://www.poemtoday.com/poems/3568?keyword=plato)

[PoemToday](http://www.poemtoday.com/) (also on [Github](https://github.com/alexpatriquin/poem-today)) is a Rails web app that statistically generates a poem based on its users' profile and behavior. Each of the 6,000 poems on the site actually has a link wrapped around every word in every poem. When a user clicks one of the words, the site initiates a search of its database for the best-matching poem and redirects the user to the top result, alongside with the top image from the Flickr API for that word.

```ruby
def save_top_result
  @poem = @results.inject do |winning,result| 
  winning[:match_score] > result[:match_score] ? winning : result 
  end
end

def get_image_url(keyword)
  flickr.photos.search("text"=>"#{keyword}", "sort"=> "relevance").first
end
```

Behind the scenes, PoemToday is storing information about all of the words the user has clicked and the poems the user has visited in a temporary session. When the session has enough data, it statistically generates a completely unique "ephemeral" poem with a statistical process known as a [Markov chain](http://en.wikipedia.org/wiki/Markov_chain).

###simplenlg vs. Markov chains

Initially I tried to implement a linguistics strategy for randomly generating poems. If a user had clicked three nouns, I figured,  how difficult would it be to find a verb and then create a semi-sensical sentence? Using the Wordnik API and a ruleset I could fill in the missing parts of speech.

I spent a while building a Natural Language Generator without realizing that was what I was doing. Then I stumbled on simplenlg, both the [Java API](https://code.google.com/p/simplenlg/) and the [ruby gem](https://github.com/efi/simplenlg), which opened up a whole new tangent of linguistic programming to me. I really love language (why else would I make a random poetry app?? ;) and this intersection of code and linguistics was an especially exciting turn in the project.

With user-selected keywords, simplenlg, and the Wordnik API's random words endpoint, which features random word generation by part of speech as well as frequency, I was able to weave together some fairly compelling random poems.

Then I stumbled upon Markov chains in my research, which on the first few tests in the command line quickly proved much more compelling than my patchwork NLG solution. The Markov chains were uncanny and required users click far fewer keywords to produce better sounding random poetry. It was a much lower bar to "minimum emotional product". 

```ruby
def create_ephemeral_poem
  @poem_title = session[:ephemeral_poem].values[-4..-1].join(' ')
  @poem_body << markov.generate_5_sentences
  @poet = "You"
end
```

The most fascinating aspect of Markov chains to me is that they know nothing about parts of speech or linguistics. They are just pure mathematics. And yet they sound so human.

I implemented the Markov chain with the [Marky Markov gem](https://github.com/zolrath/marky_markov), though I will do my own [implementation in ruby](https://gist.github.com/alexpatriquin/11226396) soon.

###Matching Algorithm

PoemToday also features a daily email option. The app will email you a poem every morning, along with information about why that poem was matched to you on that day.

The app connects to APIs from Twitter, The New York Times and Forecast.io for user profile data sources. The app also uses the Wordnik API to score keywords on frequency from each data source. The Wordnik API goes back to 1800, but I limited it to 2000 for contemporary word usage.

I treated keyword frequency like golf scores - the lower, the better, so that rare words percolate to the top of poem matches and users' inboxes. You can get a good sense of how the user-to-poem daily matching algorithm works from this Gist.

```ruby Keyword-Poem Match Scoring https://github.com/alexpatriquin/poem-today/blob/master/app/models/poem_scorer.rb source
  def score_by_keyword_source(result)
    case result[:keyword_source]
    when :twitter
      result[:match_score] += 40
    when :news
      result[:match_score] += 30
    when :forecast
      result[:match_score] += 20
    end
  end
 
  def score_by_match_type(result)
    case result[:match_type]
    when :occasion || :subject || :poet
      result[:match_score] += 100
    when :title
      result[:match_score] += 50
    when :first_line
      result[:match_score] += 30  
    when :content
      result[:match_score] += 10
    end
  end
 
  def score_by_frequency(result)
    #birthdays, name days and holidays have 0 freq => 100 points
    frequency_score = (1000 - result[:keyword_frequency]) / 10
    result[:match_score] += frequency_score
  end
```

I also used a Postgres search gem, pg_search, to quickly search through all of the poems and their content. This was a lightweight option versus implementing a platform such as Solr or EC2.

Finally, I used [Twilio's text-to-speech](http://localhost:4000/twilio-text-to-speech-in-rails-4) feature to read the poems in a fun robot voice. This was actually one of harder parts of the project as it required that I create a virtual [Twilio device](http://localhost:4000/twilio-text-to-speech-in-rails-4) with Javascript on each poem page load and could only test it in production or with ultra-slow ngrok, which exposed my local web server to the internet.

PoemToday was definitely my most ambitious technical project to date. Judging from the fascinated reactions of a few dozen people at the Flatiron School Science Fair, I'd say it was a success so far and I look forward to continuing to build it.