---
layout:     post
title:      Making a discord bot
date:       2021-02-16 23:39:00
summary:    I made a simple discord bot that reacts to my friends messages
categories: python
---

####Please note that this post is currently under construction - thank you.

I had the idea to make a simple discord bot that would react to my friends messages as a goof.

A 'reaction' on discord looks like this:

![](https://www.bgigurtsis.com/pictures/posts/discordbot/1.png)

The concept was to make it so it reacted when a specific friend posted an image or a link in chat. Here's my full code:

{% highlight python %}
# bot.py
import os
import re
import discord

from dotenv import load_dotenv

load_dotenv()

client = discord.Client()

@client.event
# function to check for regex matching a url
async def on_message(message):

    emoji = '<:timeout2:806320653451001856>'

    regex = re.compile('((http|https)\:\/\/)?[a-zA-Z0-9\.\/\?\:@\-_=#]+\.([a-zA-Z]){2,6}([a-zA-Z0-9\.\&\/\?\:@\-_=#])*')
    match = regex.search(message.content)#

    if message.author.bot:
        return
    elif message.author.id == 389104618099048468:
        if match or message.attachments:
            await message.add_reaction(emoji)

client.run(os.getenv('TOKEN'))
{% endhighlight %}

{% highlight python %}

{% endhighlight %}


I'll take you through the code bit by bit, explaining what each part does.

{% highlight python %}
load_dotenv()
{% endhighlight %}

This calls load_dotenv, loading environment variables from a file called .env in the current directory. My .env file holds the OAuth2 token that allows the bot to connect with my Discord developer account.

It was important to put this file .gitignore if publishing code to GitHub as anyone with this token could place their bot in my Discord server with admin privileges. I did this accidentally. Luckily for me Discord itself has a bot that crawls GitHub and alerts you if you upload your token there by accident - pretty cool!

{% highlight python %}
client = discord.Client()

@client.event
{% endhighlight %}

Create an instance of 'client', our connection to Discord. The second line tells python that an even needs to happen in order to execute.

{% highlight python %}
async def on_message(message):
{% endhighlight %}

Runs the following code when a message is received

{% highlight python %}
emoji = '<:timeout2:806320653451001856>'
{% endhighlight %}
