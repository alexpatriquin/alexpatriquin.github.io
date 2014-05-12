---
layout: post
title: "Introducing Stackup"
permalink: "/stackup-angellist-for-tech-stacks"
date: 2014-05-12 13:01:43 -0400
comments: true
categories: [Projects, Rails 4, NLP, OAuth, Redis]
---

[{% img center ../images/post_images/crowdsurge-serp.png "StackupApp" %}](http://stackupapp.herokuapp.com/)
## Description 

[StackupApp](http://stackupapp.herokuapp.com/) is AngelList for tech stacks. The app seeks to map out the current technology landscape of the startup ecosystem. The app injests startups from the AngelList API and tags each startup with programming language and tech stack keywords from Indeed job postings. 

Currently there at 5,318 New York City-based startups on StackupApp, of which 693 have been tagged.

## Technical Features

##### APIs and OAuth

In addition to retrieving startup information from the AngelList API and job postings from Indeed's API, StackupApp also enables users to sign in via AngelList OAuth. [The OAuth implementation](https://github.com/alexpatriquin/stackupapp/blob/master/app/models/user.rb) relies upon Devise and OmniAuth.

If the user is a Founder of a AngelList startup also on StackupApp, they will see a Manage section on the startup's page and their profile page. The Manage section enables the user to Edit the startup's tags (under development), Unclaim or Deadpool the startup.

If you don't have an AngelList profile or would like to use a demo account, you can sign in via AngelList OAuth with the StackupApp test account credentials (hello at poemtoday dot com, abcd1234).

[{% img center ../images/post_images/manage.png "Manage" %}](http://stackupapp.herokuapp.com/)

##### RESTful Routes

StackupApp is built with Rails, and takes full advantage of the framework's REST conventions. The Unclaim and Deadpool buttons intiate DELETE actions on the [UserStartups](https://github.com/alexpatriquin/stackupapp/blob/master/app/controllers/user_startups_controller.rb) and [Startups](/app/controllers/startups_controller.rb) controllers, respectively, and can only be access as an authenticated user.

##### Backgrounds jobs with Sidekiq and Redis

When a user has signed in via AngelList, the app makes an API request to AngelList to retrieve the user's startups. This request is made via a background job to improve performance, specifcially via [a Sidekiq Worker ](https://github.com/alexpatriquin/stackupapp/blob/master/app/workers/angellist_worker.rb) and Redis.

##### Scopes and counter_cache

The app displays startups and technology tags by popularity scopes. Tag popularity is determined by counting the number of [StatupTags](https://github.com/alexpatriquin/stackupapp/blob/master/app/models/startup_tag.rb) or job postings with each tag. A ```counter_cache``` is used on the StartupTag table to improve performance. The AngelList API provides ```follower_count``` which serves as the startup popularity metric.

The app enables users to ```search_by_name``` for startups or ```search_by_tag``` for tecnologies, which is implemented with the ```pg_search``` gem on [the Startup model](https://github.com/alexpatriquin/stackupapp/blob/master/app/models/startup.rb). Only startups that have been tagged are shown in the index and search results, while all startups are searchable by name.

## Continuing Development
Continuing development plans include moving the frontend into Ember.js. Also, expanding to SF and Boston.