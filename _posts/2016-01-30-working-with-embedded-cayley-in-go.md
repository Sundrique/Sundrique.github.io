---
layout: post
title:  "Working with embedded cayley in Go"
tags: [go, cayley, google, gophergala, graphs, graph database]
date:   2016-01-30 19:15:00
image:
  feature: cover.jpg
---

About one month age I helped out my colleage to realize his idea on gophergala2016. The idea is to boost productivity of individual team members and overall teams as well by using team pomodoro timer.
Being both Go newbies we wanted to use Go written tools and libraries as much as possible.
We decided that graph database would fit perfectly our needs.

We failed to find the comprehensive documentation of using Caley as embedded graph DB. Although this article doesn't cover all the aspects of Calye it will give you some clues on using it's Go API.
Let imagine the following Scenario: users, groups, users can follow each other, if users are in the same group

##Add quad

Alice follows Bob
Alice follows Charly
Bob follows Charly
Charly follows Alice
Charly follows Bob

Bob in Group1
Alice in Group1
Alice in Group2 

Alice posts Post1
Alice posts Post2
Bob posts Post3
Charly posts Post4

##Outgoing relations
V('Alice').Out('follows')

##Incoming edges
V('Group1').In('follows')

##Complex
V('Alice').Out('joins').In('joins').Out('posts') //can we remove alice

##Join
Join direct following with group following

##Remove Quad
