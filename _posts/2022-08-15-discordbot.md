---
layout:     post
title:      Making a simple Discord bot with Python
date:       2022-08-15 23:39:00
summary:    Discord python bot that responds to a certain user's messages.
categories: engineering
comments: true
---
I had the idea to make a simple discord bot that would react to a user's messages. The bot uses [discord.py](https://discordpy.readthedocs.io/en/latest/index.html), a Discord API wrapper built in python. It is designed to react if, and only if, a specific user sends either a link or uploads an image.

A 'reaction' on discord looks like this, nothing too intrusive:

![](https://www.bgigurtsis.com/pictures/posts/discordbot/1.png)

Here's the full code for my bot:

```python
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
```

I'll take you through the code bit by bit, explaining what each part does.

```python
load_dotenv()
```

This calls <code>load_dotenv</code>, loading environment variables from a file called `.env` in the current directory. My `.env` file holds the OAuth2 token that allows the bot to connect with my Discord developer account.

It's important to put `.env` inside `.gitignore` if publishing the code to GitHub as anyone with this token could place their bot in my Discord server with admin privileges. I did this accidentally. Luckily for me, Discord has a bot that crawls GitHub, cancels your token and alerts you if you upload your token there by accident. Thanks, Discord!

```python
client = discord.Client()

@client.event
```

Create an instance of `client`, our connection to Discord. The second line tells python that an event needs to happen to execute.

```python
async def on_message(message):
```

Runs the following code when a message is received.

```python
emoji = '<:timeout2:806320653451001856>'
```

I want the bot to react to messages with a specific emoji. Here I define which emoji and ID of that emoji.

```python
regex = re.compile('((http|https)\:\/\/)?[a-zA-Z0-9\.\/\?\:@\-_=#]+\.([a-zA-Z]){2,6}([a-zA-Z0-9\.\&\/\?\:@\-_=#])*')
match = regex.search(message.content)#
```

These two lines of code are how the bot detects links in a specific message. I define some regex will only match to a link and pass this as a function `match`.

```python
if message.author.bot:
    return
elif message.author.id == 389104618099048468:
    if match or message.attachments:
        await message.add_reaction(emoji)
```

This block first checks if the message received is from any bot. If so it returns and waits for another message.

Otherwise, it checks for a specific user ID that I define. Then if it that message either match's the regex (is a link) or if the message contains an attachment the bot reacts to that message.

```python
client.run(os.getenv('TOKEN'))
```

Finally, this code takes the OAuth2 token and passes it to the bot.

Further information and documentation for discord.py can be found [here](https://discordpy.readthedocs.io/en/latest/index.html).
