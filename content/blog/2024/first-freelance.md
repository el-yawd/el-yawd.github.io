+++
title = 'First Freelance'
date = 2024-06-05T23:05:53-03:00
author = 'Diego Reis'
tags = [
    "career",
    "experiences",
]
+++

>This's my first blog post, so if you're reading this, thank you! The idea
>is to write regularly, documenting a bit about my experience as a Computer Science
>student getting into the job market.

Let's get straight to the point: how my first freelance went and what lessons I learned.

## Freelance Definition

> "Freelance is all, closed-scope, paid work where there's no employment relationship between the payer and the one getting the job done."                          -- Me


### Some Context

I started learning web dev nearly a year ago, following the usual path: HTML, CSS,
jumping to vanilla JS, then to React (which I later regretted — learning about 'addEventListener' and other stuff first
would’ve been helpful), Nodejs,  SQL, hot to develop basic REST APIs, a little bit of docker and most recently Java.


Each project helped me to improve at some aspect: speed (in terms of delivery, not raw performance), maintainability,
learning completely new stacks. I learned things on-demand and continue to live my life. In some of these projects I met
a pretty nice guy who guided me to other roads of best development practices, the ~infamous~ clean code, and refactoring.

This particularly project was a v2 of another project that this guy worked on. As the v2 still in a development phase,
the v1 has been used and it about it that we gonna talk about.


### Problems

This project is a contest study platform, developed in Vue.js, PHP/Laravel and MySql (stacks that I never worked on).
And it was having two major issues:

1. A field that should highlight at a specific date was not working;
2. Some options that should appear after a specific date was not showing up;

Simple don't you think? That's was the two bugs that I was paid to fix.

## Solutions

Unfortunately I can't show the source code as I like but basically the first problem as just a miss object constructor
related with `moment.js`, the second was a trickier one, some obscure SQL query had a wrong conditional.

But how I came to these solutions? What I did wrong? I most important, what I've learned?

## Learnings

### 1. Focus on the problem

With no doubts my biggest mistake was trying to understand all the system at once, instead of focusing exclusively
on the problems.

Working with other people code it's a funny thing, and surely the most common scenario, you have to understand the _how's_ and _why's_, explore components, modules and classes in such way to create a mental model of the system the most
close as possible of the previous developer. Doing this it's not easy and can take days or even months, but I wont have too much time. When I realized that I was trying to understand the code of each page instead of focusing on the buggy page I pivoted.

With specific tasks comes specific approaches. In reality is much simpler than it seems, there is no need to refactor
all the code, just need to fix this tiny problem. What bring us to the next learning.


### 2. Create and test hypothesis

A practical lesson. The scientific method[1] applies exceptionally well in debug sessions. For instance,
a system as some unexpected behavior, after digging into the logs, requests, and any trail of the bug, you:

1. Create a hypothesis about the reason of the current behavior;
2. Create a test scenario that makes your hypothesis fail;
3. Repeat;

I particularly have a great time doing this kind of things, I feel like a crime detective in the crime scene, or more
precisely a mechanic fixing a car already "fixed" by another mechanic. Create hypothesis, test, repeat, and be curious. At least you going to learn something new, and don't forget:

> "When you have eliminated the impossible whatever remains, however improbable, must be the Truth"
>                                                                           -- Mr. Spock


### 3. Don't be afraid of new stacks

As you could see about my background, I've never has worked with any of the project techs, but besides these techs:

- The HTTP protocol is the same;
- The _front-end -> back-end -> database_ model is the same;
- Laravel's structure with controllers, services and models is not too different of other web frameworks.

Mainstream techs are more similar than we want to admit. So, if you learned the problem that a tool proposes to solve,
you can understand variations of such tools, and the different was that these variations some the same problem.

### 4. Take risks!

Last but not least, take risks! I'm saying to bet on shitcoins? No! Or lend money to your relative open a snake oil store? ~Maybe~ NO! I'm saying that putting yourself in situations that you have nothing to lose, but a lot to gain is always good[2], even if you think you're not 100% ready. Each opportunity that takes you off your comfort zone is a opportunity to improvement.


#### Notes

1. A powerful concept, even routine in programming, is the Ockahm's razor, that says: "[...] The simplest explanation is usually the best one". So, before checking for reace conditions or deadlocks, certify about that semicolon on line 67 :)
2. For those who read Taleb's books that's the bread and butter of _antifragility_, I recommend any of Nassim's book.
