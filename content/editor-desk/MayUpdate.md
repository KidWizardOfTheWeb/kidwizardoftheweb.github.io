+++
date = '2026-05-09T14:44:14-04:00'
draft = false
title = 'May Updates'
+++

{{< toc >}}

Summer is here in the Northern Hemisphere. My office upstairs gets extremely hot some nights, but that's how you know you do real work.

## New updates for Limiter Cut Radio

Development has been moving along smoothly, where I've ended up adding more features and creating a linux server instance that anyone can connect to right now! I've learned a lot about Debian Linux and how to spin up a VPS to host the server script as a live service.

While we did use Linux in college, that was mostly GUI based (I forgot the distro, too). Here, I learned how to handle everything from systemctl services to package management, and even how to host your own domains with SSL certificates. It's been a lot of fun learning all about it, I've even considered moving my existing websites off of cloudflare and to my server at some point to self host. For now, let's talk about the new features.

### Overall code refactor: add/remove async services.

When adding the services talked about below, I had to refactor the client-side code to multiple files. Originally, I wanted a single client script and a single server script in order to make setup/engagement as simple as possible for users in case they clone and run the source code. But adding too many async options to wait on in a single script became hard to read and became dysfunctional when all the possible states were checked for. Therefore, the new model is like so:

```Py
| clientDriver.py # <- This is our main driver. It imports all of our other services as needed, and gathers them to run asynchronously.
|-> webSockClient.py # <- Our voice chat/radio functionality for the client. This task connects to our server and then transfers/receives audio packets as needed.
|-> redisClient.py # <- Our Redis instance activities. Currently this is relegated to chat functionality, but notifications are in the plans to be added.
|-> # Add services as needed...
```

This new setup means that users still only have to run one script, while developers can add services that aren't directly attached to each other by importing and running async in the driver. Cleaner development structure means faster deployments.

### New feature: text chat with Redis Pub/Sub

What's more is that I have learned the power of Redis Pub/Sub in order to create transient chats (similar to Twitch without the storage currently). When a channel is spun up, a channel is created and subscribed to with the same name as the original channel. Other users who connect are automatically subscribed to the text chat as well. This part needs a little cleanup for the desktop version, but is entirely functional. This should make interactions between hosts/chatters easier overall, and is a lightweight enough addition to include.

### New feature in testing: voice chat.

That's right, now LC-Radio has an implementation to be used as a voice chat client as well! Previous tests to make this work in an async environment ended in complete disaster (using the QUIC protocol). After switching to Websockets and learning more about applying coroutines effectively, a simple pub/sub method for audio packet transfer was created where everyone in a "room" would broadcast their packets to others who are in the room. The `broadcast()` command in the python Websockets library made this even easier to perform. The server simply redistributes packets as needed, with all processing incoming/outgoing handled on the client side.

The only issue that happens here is with mixing audio. And it's a major one.

At a surface level, the process for a voice chat looks something like this to the average person:

```
Client <-> server <-> Client
```

Zoom and some other services handle things this way, to varying degrees of quality. But to reach something like discord, that's a different story.

It is NEVER that simple in reality, especially for something that has to account for a highly scalable amount of users, connection quality, avoiding overflowing send/receive buffers, etc.

Discord has a [slightly older yet still relevant article](https://discord.com/blog/how-discord-handles-two-and-half-million-concurrent-voice-users-using-webrtc) on how they did their solution, but for a regular person to attempt to replicate such a deep system is almost futile. 

In reality, voice chat is something more like this (at least, for my case):

```
One-way example (send to receive)

Sending client(s) -> server -> receiving client -> mixer (for all incoming audio stream data) -> audio device
```

The issues become a little more apparent here. Having multiple audio streams incoming and having to mix them together to create coherent output is the biggest blocker here. How does one mix streams that could either send data in asynchronously or have different timings? 

Mathematically, this would be possible if all streams were sending data constantly. However, we do not want that, as that overflows the buffers and creates too much traffic for the websocket to handle as many connections as possible.

The logical solution for those who have done this before would be to use WebRTC as a framework with an SFU for scaling or an MCU instead of working with WebSockets in this manner. WebSockets are more for signalling, such as being used for signalling in an WebRTC solution.

While I have attempted to use WebRTC, there still wasn't a very clear solution to handling audio mixing from multiple incoming streams that I could find. So for now, I'll have to do more research into a solution like this. I have to rewrite both the server and client code implementation to even handle this method. It's honestly very complicated from what I've been reading and not very straightforward to just read and implement either, you have to have been using it from the start. I wish there were better examples that weren't just javascript/typescript based. Discord does use WebRTC in C++ and has their own SFU implemented, but it is closed source so you can't really garner what was exactly created.

MCUs are designed to mix audio and video streams on the server itself, but is computationally expensive and isn't for a program like LC-Radio's usecase for that reason. Our goal is to allow multiple channels to be available to all, and for a feature that wasn't intended at the start such as voicechat, it would be a lot of extra power for a secondary feature. 

While an exciting prospect, for now we'll be keeping voice chat on the backburner. 


## New project: RidersPyTools

In short, this is a python package/framework that I've been theorizing for a long time, and I've finally developed it into a working, extendable version!

The explanation is a bit too long for this article alone, so check out the Making of RidersPyTools post for more information!