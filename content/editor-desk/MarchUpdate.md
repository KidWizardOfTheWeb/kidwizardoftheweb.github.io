+++
date = '2026-03-16T13:30:25-04:00'
draft = false
title = 'March Project Update'
+++

A bit has been going on since January (the January post was collecting dust on my local in fact). 

### Latest main project: Limiter Cut Radio

This was an idea that I've had for the longest time: creating my own "radio" service. The basic idea is that anyone could connect to a "station" and start hosting their own audio, simple as that.

I was mostly disappointed by internet radio, being pre-recorded and canned audio for the most part where music streams on a schedule. While that's cool and all, I'd definitely appreciate the feeling of a live radio host being there and being in complete control of what one can stream. Back in college, I joined the college radio for a short time and I was enamored by hosting. While I never had the chance to host a show on my own, it was something I dreamed about still. That feeling carried me into developing this project.

Simply put, no accounts are required here. You can connect to a server, host your own channel, and any user can listen in to the audio you stream. That's it. No extra video or subscription shenanigans, purely radio at it's finest for anyone to use. 
It is currently in a simple state right now, but functional locally with no reason why it could not work on a server. Here's how it works.

### The workings and structure of LC-Radio

With LC-Radio, users can either cast audio 1-way or listen to casters stream audio.   

For casters:
1. Start up the app, connect to the server.
2. Request a channel that does not exist or is empty. A channel can be ANY ID (currently) that is a valid string.
3. The server connects assigns you to the channel you chose and you start streaming audio packets to anyone who connects to the channel afterwards.

For listeners:
1. Start up the app, connect to the server.
2. Request a channel that exists already. 
3. The server connects you to the chosen channel and you receive audio packets from the host. You can drop in and out at any time without disrupting.

Here's a video from the early demos as an example. 

**Note**: The drop-out bug crashing the server has been removed since this, server is self-sufficient and allows casters/listeners to come and go as they please with no issues. 1 second delay was included to show off the audio echoing in this local test.

{{< video
  src="/mp4/LCRadioDemo1.mp4"
  width="800"
>}}

It's a very simple setup with a lot of room for creativity on the caster's part to do any type of show they want, be it a podcast or streaming their own music. I see this as a return to form for the early 2000s. Not as much for nostalgia as much as it's for reducing the amount of noise streaming services have in terms of ads and features.

While I have considered two-way communication, the aim of this is a modern version of radio without as much restrictions. Services like mumble already exist and are fantastic for what they do, but I have never seen something done like this before.

Features to be added include multi-casting with a host-caster and co-casters for shows, and a chat for listeners to interact with casters. I will also try to make a GUI like I did with RCMM (I really enjoyed doing that and how it's grown), and attempt to make a webapp/mobile variant after the desktop app is confirmed to work. Next up is to tighten up the data structures and host on an EC2 instance first, since this will most likely need a large-scale stress test eventually.

As are most of my projects, everything is open sourced for any contributions. The API will also be open-sourced in case anyone wants to host their own servers (which I hope they do, that would make me happy).

The repo can be found here and will be updated periodically:
https://github.com/KidWizardOfTheWeb/LimiterCut.Radio

Do note that the rest of the month is probably going to be a short break from this, as I'm doing a major project on machine learning for a course right now and that takes priority. I'll do a formal explanation on the structures later when I create proper classes for everything.

### The job search, again.

It has been getting a bit better, more opportunities are out there but I'm doing a lot of waiting on responses these days... referrals are really the only way to go. Here's hoping to finally landing something, I've been to the last interview and lost out more times than I can count.

### What's next?

Finishing LC-Radio and making it Reaper Co.'s first official non-game project. It's nothing on the level of major applications, but I hope the more niche/decentralized services people pick up on it.

I do want to get back to working on games, but I haven't felt so passionate about something non-game related in so long that it might be a while. Games industry is still kind of stuck in a bad place along with the rest of the market, but I won't quit making things, because what else is there to do in this chaotic world except keep pushing on.