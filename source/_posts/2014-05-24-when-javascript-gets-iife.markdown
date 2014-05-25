---
layout: post
title: "When JavaScript gets IIFE"
permalink: "/when-javascript-gets-iife"
date: 2014-05-24 15:47:29 -0400
comments: true
categories: [JavaScript]
---

Recently, while working on a JavaScript project, I experimented with putting a closure a loop. It did not work, and for reasons I couldn’t initially explain. 

After a good bit of research, it turns out I had stumbled into a well-understood issue (some call it a bug) known as “fold over.” In this post, I’ll go over what “fold over” means and a few alternatives for counteracting it, including a very useful design pattern known as an Immediately Invoked Function Expression (IIFE). 

###Background: A Game of Blackjack

The project was to build a simple version of the game of Blackjack in JavaScript. I had set up the game such that each round could have any number of players and rounds would end with an array of each player and their card total (with face cards worth 10 points and aces worth 1 or 11).

<!-- more -->

{% codeblock %} var round1 = [{name:"Player1", cardTot:18}, {name:"Player2",cardTot:21}, {name:"Player3",cardTot: 24}]; {% endcodeblock %}


A `score` function would then accept the round as an argument, as the `hands` parameter. Then a for loop held a closure (or function inside a function) that checked to see if `cardTot` was over 21 and, if not, return the player’s blackjack score for that hand. Obviously, in blackjack, if your hand goes over 21, you bust and get 0 for the round.

``` javascript
function score(hands) {
    for (i = 0; i < hands.length; i++) {
        hands[i].bjScore = function () {
            if (hands[i].cardTot > 21) {
                return 0;
            } else return hands[i].cardTot;
        }
    }
    return hands;
}
score(round1)[0].bjScore();
``` 

The issue arose on line 11 when I tried to call `bjScore();` on Player1. 

{% codeblock %} TypeError: Cannot read property 'cardTot' of undefined {% endcodeblock %}

The error was that `cardTot` was getting called on `hands[3]`, which did not exist. The zero-based `hands` array was only 3 players long. How did this happen?

FYI - A simpler and probably better approach would have been simply to assign `bjScore` for each player’s hand, instead of setting up the closure to do so later. But, as mentioned, I was just experimenting.

###Understanding How Loops Can “Fold Over” Closures

A closure has access to the variables defined within its own scope, as well as the variables and even parameters of its outer function(s). However, a closure does not store the actual values of its outer functions. A closure only stores references to them. 

Moreover a closure refers to its outer function’s variables even after the outer function has returned. In the context of for loops, this post-return access creates the “fold over” effect. Left at default access, closures will always call the last value of the iterator in the loop.

For example, in the `score(hands)` example above, the for loop has fully executed before the `bjScore();` closure ever gets called. Thus, the value of i on line 10 is hands.length or 2. And when `bjScore();` does get called on the next line, doing so actually increments i one more time, so that that a `TypeError` is raised. 

Even if calling `bjScore();` did not increment `i`, the value of `i` at the end of the for loop would still be 2, which would prevent us from retrieving any of the actual blackjack scores for players other than Player3, who’s at the 2-index. In other words, we would get the final player’s `bjScore` no matter which player we called `bjScore();` on.

###Immediately Invoked Function Expression (IIFE) to the Rescue

One way to counteract “fold over” in loops is to use an [IIFE](http://benalman.com/news/2010/11/immediately-invoked-function-expression/), which is a design pattern that saves the state of a variable value so that it can be passed into a closure. Usually a closure would have to accept post-return access to a loop’s final value, but with IIFE, we can “lock in” and pass the value into the closure at each turn of the loop. 

``` javascript
function score(hands) {
    for (i = 0; i < hands.length; i++) {
        hands[i].bjScore = (function () {
            if (hands[i].cardTot > 21) {
                return 0;
            } else return hands[i].cardTot;
        })();
    }
    return hands;
}
score(round0)[2].bjScore;
``` 

Here is the `score(hands)` function again, but now the closure as an IIFE. The closure gets wrapped in parentheses so that Javascript will evaluate it as a function expression and then that function gets immediately invoked with the current value of i. The setup is easier to see without the example code. 

{% codeblock %} (function () { /* code */ }()); {% endcodeblock %}

Note also that we don’t need to call `bjScore();` as a function any longer since the IIFE created `bjScore` as a property on each player object. Now we can just call `bjScore` and get back the correct value for each player.