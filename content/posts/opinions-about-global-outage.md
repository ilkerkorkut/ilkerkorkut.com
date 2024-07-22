---
title: "A candid subjective consideration about Global Outage"
author: ilker
date: 2024-06-26
tags:
  - microsoft
  - crowdstrike
  - opinions
  - sdlc
categories:
  -  consideration
keywords:
  - global outage
  - sdlc
  - c++
  - rust
---

Firstly, thanks god, I am using Windows for only playing games : ).

I heard the outage after nearly one day later. When I took a look about the issues about what is going on, too much different information is there. I have to confess that I really surprised about the incident. I can’t suggest any solution or kind of approach only I try to criticize it and ask questions while writing this post.

I can start with the Information I found while searching generally points followings;
- Invalid memory region
- Null pointer dereference
- Out-of-bounds memory access
- Permissions or exploitation


In user(consumer/customer) perspective, those kind of stuff does not mean anything for people.


Why I am writing this post? I needed an electricity technician for my home to fix something. While we are talking he said that; “something will happen, did you see there was an attack or someone intentionally downed the digital things.”. At that moment, I realized that how can I explain the details about that global outage? Nearly, everybody knows about Microsoft Windows, but there is something runs on the kernel on windows. I was still thinking about the explaining about something is working on core of the windows. Then I realized that, he can understand when I say there was an application is running behind the scenes on the windows and that is another company’s software. I think he understand the basics. But, the problem I was still thinking about the incident, who can allow that any kernel dependency may cause completely down.

In daily basis, we are talking about availability, robustness, sustainability, reactivity etc. with the engineering team members. Also, single point of failure problems are nearly stays in first place.  It’s quite weird billion dollar companies may have those kind of issues. I think the perfection is a myth while managing a project, even though, there are several prevention techniques or workflows to handle that.

As far as we see, financial systems, airport, and governmental systems are the most damaged systems and they affected the people’s usual life. Without thinking internet deprivation, what happens if our systems could use the cloud based os? Most of the solutions are based on the manual guidelines such as running Windows with safe mode and removing some files etc.

What could happen if any currency exchange offices, atms, airport displays etc. used like a cloud-based operating system? 

We know, those two kind of business highly depended on the internet connection, so that it is quite feasible to use kind of stuff. Also, we can assume cloud based os can be ran under a intranet like networks as well. Under that condition, any update would have immediate rollback without any human interaction. And it can cover the businesses and the people who would like to use that business services. At worst case, partial outage could be happen, as a end-user we can assume the cloud would fix itself in a timeframe. We(software engineers) already know most of the tech businesses has site reliability engineering teams for that purposes.

I believe that, in the future companies will highly think about that when it’s quite feasible to use(today’s most important issues are, continuous internet connection, low-speed of the connection) any cloud based os when they are existed.

I don’t believe the conspiracies. However, sometimes it may be quite reasonable if you want to control the crowds. I consider that humans are still primitive creatures somehow. Of course, some of minorities are not inside that crowd. Actually still, anybody can convince people to something like a wizarding actions like the older dates. In our world, that corresponds to mainly digital environment. Because it’s like a magic in the eyes of the crowd and that works somehow. We can derive conspiracies as well, for instance, in any country(mostly second or third world countries) internet can be controlled by the governments and their digital warriors can inject something inside the our machines right? This is quite reasonable and technically feasible. So if we believe that, any kind of government can push the button for any outage or any incident like a nuke. Is there any escape situation, of course not. In my opinion, people shouldn't obsess that kind of things.

Forget all that stuff, most common opinion(and mine) is this is obvious management issue, also anybody can’t blame the company who is responsible for. Because there is a homogenous environment inside the machines and acting like a monolithic system. Changing a programming language for that purpose is just like blaming only the development phase and the developers. I see there are lots of suggestions about `c++` to `rust` migration to achieve more reliability like preventing mistakes. But it’s much more blaming the developers, not the actual workflow itself.

I believe this will led to new approaches and the architecture designs and obviously, this incident’s echoes will survive. Lots of companies would reconsider about their approaches and try to find new solutions to manage kind of dependencies inside their applications or systems. Thanks for reading.