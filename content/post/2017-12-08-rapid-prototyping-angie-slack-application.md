---
title: Rapid prototyping & Angie & Slack application
date: 2017-12-08T10:58:51+01:00
author: zrzka
layout: post
tags: [prototyping]
---

We're trying new ideas for our [startup](https://www.purposefly.com) and they need to be tested quickly.
With real users. We're just connecting available dots without reinventing the wheel if there's
existing solution for particular problem. Here's an example what we just did in a few days.

Our flagship [Frank](https://www.purposefly.com/frank/) is well accepted by customers. They like it,
because it's simple (survey with five questions) and it gives them good results. When compared to another
performance review tools (like 360, etc.), it's blazingly fast. But how many surveys are you going to
create per year? Three, four? Probably. We're still thinking how to improve the whole evaluation
process and here comes another idea - Angie - it's all about results and fair evaluation. Fair means
that team members are going to evaluate each other, not biased manager. This evaluation is processed
more often and is based on events. Event can be anything in this context. For example - daily meetings,
releases, business deals, ... Simply whatever. Every team can decide how often they would like to
evaluate each other. Every team member can create an event. Team members gain / lose reputation
points based on actions like create event, up vote, down vote, ... Team leader (or all members)
can decide what they're going to do with these results - it can be part of your salary, one-time
rewards, etc. This tool is also inspired by the
[Stack Overflow reputation system](https://stackoverflow.com/help/whats-reputation).
Let's write some prototype.

## Teams, users

First thing we need are users and teams. The easiest way is to reuse existing information. Don't
forget, we're just prototyping. Lot of companies use [Slack](https://slack.com) today. Slack has
quite nice API and interaction components like dialogs, interactive messages with buttons, ...
What about teams? Team in the Slack context is the whole organization. But there're channels
and these channels are mainly used to divide people into groups with some specific topic. Angie
can treat channels as teams for now.

Nice, first task done - we have [users](https://api.slack.com/methods/users.list) and
[teams](https://api.slack.com/methods/channels.list). What's important is that no one needs to
register, create organization, maintain members, organisation chart, ... It's really important
for prototyping. Not just for you (developer), but also for your prototype users / testers.
Keep it as simple as possible.

## Create event

Event is an entity with name and list of participants (= team members = Slack's channel members).
Slack provides [dialogs](https://api.slack.com/dialogs). They're kinda limited (text field, text
area or options), but they suit our needs and we are going to use them. How one can open a dialog?
[Slash command](https://api.slack.com/slash-commands). In our case, it's
`/angie new [event-name] [#channel]`. Here's the result.

![Angie - New event](/images/purposefly/slack-angie-new-event.png)

Whenever user hits _Create_ button, application receives payload with validated user input. We can
validate it again (if it's required, Slack provides simple validation like input length only), send
back errors (shown in the dialog itself) or just accept it for processing.

## Notification to users

We can think of [ephemeral messages](https://api.slack.com/methods/chat.postEphemeral) sent to
respective users in the selected channel. It's not a good idea.

> Ephemeral message delivery is not guaranteed — the user must be currently active in Slack
> and a member of the specified channel. By nature, ephemeral messages do not persist across
> reloads, desktop and mobile apps, or sessions.

I already did this mistake, because I missed this paragraph in the documentation and these
messages are not delivered if user is not active (offline, ...). We stick with
[im.list](https://api.slack.com/methods/im.list), [im.open](https://api.slack.com/methods/im.open)
and [chat.postMessage](https://api.slack.com/methods/chat.postMessage) to the IM (direct message)
channel. These messages are always delivered.

## Voting

Any message can contain [attachments with buttons](https://api.slack.com/docs/message-attachments).
Let's reuse them. Here's the example of the voting message.

![Angie - Vote message](/images/purposefly/slack-angie-vote-message.png)

[Buttons](https://api.slack.com/docs/message-buttons) can have style like default (black), danger
(red). Whenever user hits the button, application receives payload with callback id and other values
we did provide. Application can just accept the payload or return modified message. We're returning
modified message reflecting current status:

* black button - user didn't vote for this member,
* red button - user did vote for this member.

Voting solved as well. We know it doesn't scale for massive amount of users (message cluttered
with lot of buttons), but we also do not target huge teams, so, it's enough. Again, just prototyping.

## Additional commands

Another commands (like `/angie list`, `/angie top`, ...) were introduced as well. These commands
allow anyone to check current events statuses.

`/angie list` example:

![Angie - List command output](/images/purposefly/slack-angie-list-command.png)

`/angie me` example:

![Angie - Me command output](/images/purposefly/slack-angie-me-command.png)

## Charts

These textual reports are nice, but it's not what our users expect. Charts are simply a must
and we do not want to do develop them as well. Here comes the [plot.ly](https://plot.ly) service.
We can prepare charts in the [Jupyter Notebook](https://plot.ly/python/ipython-notebook-tutorial/)
and reuse the same code in our application with [plotly](https://pypi.python.org/pypi/plotly) package.
Just replace `plotly.offline` (notebook) with `plotly.plotly` (application). It also allows us to
upload secret charts and share them with anyone (private link with auth key). Or we can just embed
them anywhere with the same link.

New commands `/angie report` and `/angie evolution` introduced. Plot.ly is little bit slow
(takes up to 10s to generate the link), so, both of them are asynchronous. User will receive
link when they're ready.

![Angie - Links response](/images/purposefly/slack-chart-links.png)

Report leverages Sankey with all the events on the left side and members on the right side.

![Angie - Report chart](/images/purposefly/slack-report-command.png)

Evolution is a simple day to day distribution of reputation points in your team.

![Angie - Evolution chart](/images/purposefly/slack-evolution-command.png)

And now we have charts for just $400 / year. Yes, we have to pay to get secret charts with
private links. But it's still very small amount compared to other solutions like developer's
salary working with D3, ...

## What's next

Prototype is finished and we're gathering data. Later we have to analyse them, modify reputation
points distribution (change gain / lose points per action). And if we will be happy, we're going
to improve it and create MVP from it.

Glad we have so many services around these days, lot of them are for free and they do allow
us to rapidly prototype. This one was finished in several days (~ 2 weeks). Not bad for something
users can directly test.

If you'd like to test it, feel free to
[add it to your workspace](https://slack.com/oauth/authorize?client_id=16506319910.277314084213&scope=commands,bot).
**But remember, it's a prototype**.

## Other technologies used

* [Sanic](http://sanic.readthedocs.io/en/latest/),
  [aiohttp](https://aiohttp.readthedocs.io/en/stable/),
  [aiomysql](http://aiomysql.readthedocs.io/en/latest/)
* [Docker](https://www.docker.com),
  [AWS EC2](https://aws.amazon.com/ec2/),
  [AWS Aurora](https://aws.amazon.com/rds/aurora/)

That's it for now. Don't reinvent the wheel and reuse everything you can find for your prototypes,
MVPs, ...
