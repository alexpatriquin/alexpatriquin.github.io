---
layout: post
title: "Introducing Tender"
permalink: "introducing-tender"
date: 2014-04-27 17:49:57 -0400
comments: true
categories: [Projects, Rails 4, Bitcoin]
---

[Tender](http://www.tendermessenger.com/) is a social payments app (think [Venmo](https://venmo.com/)) for Bitcoin. Users can send or receive any amount of Bitcoin, including extremely small amounts which can be used to validated the identity of counter-parties.

[{% img center ../images/post_images/tender-screenshot.png 500 %}](http://www.tendermessenger.com/)

I built Tender (also on [Github](https://github.com/alexpatriquin/BitcoinMessenger)) with [Chris Kohlbrenner](https://twitter.com/CKohlbrenner). I came up with the initial idea for the project as a Bitcoin dating site... hence the triple-play on words with the name "Tender". However, Chris and I decided to generalize the app and make it usable (or at least instructive) for other open source projects as a general social messaging utility. 

###Why Bitcoin Messaging?

Bitcoin gets a lot of hype these days, but after working with the cryptocurrency a bit, I really do see its potential to transform online payments and much more. For example, a messaging system can require a very small amount of Bitcoin get sent with each message as a way to prevent or at least deter bulk volume spammers. After practicing our web development skills, our main purpose in building Tender was to demonstrate this anti-spam concept.

We used Ruby on Rails as the framework and built the Bitcoin transactions features with the Coinbase API and the messaging functionality with a gem called Mailboxer. In the process of weaving those two features together, we learned a lot about them. Payments and messaging share the concept of "credits" and "debits" to a balance or mailbox. Kind of like double-ledger accounting, we had to do a credit and a debit whenever a user sent money and a message to another user, and vice-versa. You can see that code play out in code below.

```ruby Tender Send-Request Bitcoin https://github.com/alexpatriquin/BitcoinMessenger/blob/master/app/controllers/conversations_controller.rb source
def send_message_and_bitcoin
  current_user.send_message(@recipient, @message, 'You\'ve got Bitcoin.')
  current_user.send_bitcoin(@recipient, @amount)
  @message.update(@amount, params[:send_or_request], @recipient.id)
  redirect_to @current_page, notice: "Sent #{@amount} to #{@recipient.email}."
  debit_balance(@amount)
  credit_balance(@recipient, @amount)
end

def send_message_and_request_bitcoin
  current_user.send_message(@recipient, @message, 'You\'ve got Bitcoin.')
  current_user.request_bitcoin(@recipient, @amount, @message)
  @message.update(@amount, params[:send_or_request], @recipient.id)
  redirect_to @current_page, notice: "Sent request for #{@amount} to #{@recipient.email}."
end
```

###Coinbase and Typeahead.js

To build Tender, we used Devise and Omniauth to authenticate users, enable them to sign in via Coinbase and to validate their balances on Coinbase. The app features a lot of validations and notices to users to confirm that their transactions go through when they should and shows descriptive alerts when they shouldn't, such as when the user has insufficient Bitcoin balance.

We also made it easy for users to find each other by implementing an autosuggest user search feature with Twitter's Typeahead.js, and brought the user objects to the views with the gon gem. Finally, Tender features a robust testing suite with Rspec and Capybara, at 92% code coverage.

This was a very interesting project on many levels, from working with a new payment type like Bitcoin, to getting deep into the mechanics of messaging. I also learned quite a bit about implementing complex Javascript features in Rails. 