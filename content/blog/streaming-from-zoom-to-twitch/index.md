---
date: "2020-04-01"
title: "Streaming from Zoom to Twitch (or YouTube)"
categories: ["zoom","twitch","streaming", "youtube"]
---

Zoom meetings are great for a small number of people, and if you're willing to buy a Webinar license you can also use it for a full on webinar (although there are some interesting restrictions to webinars, like you can't have breakout rooms).

Twitch is a great streaming platform, originally used for streaming video games, its now more commonly being used to stream live coding and another technology related things that involve sharing your screen.

Traditionally you'd use software called [Open Broadcaster Software (OBS)](https://obsproject.com/) which you can compose multiple video/audio inputs into a single video that can be streamed to Twitch. This is how folks will compose their xbox, a chat screen and a webcam together in their stream.

However OBS is quite CPU hungry (I think a GPU might help) and by the time you run OBS on a standard machine its tough to run anything else without starving OBS of resources and causing stuttering on the stream.

Its made worse if you want to have multiple people on the stream and you end up running something like Zoom which allows you all to chat, and one or more of you to share your screen via zoom, then you just push Zoom itself through OBS.

But if you're already streaming video via Zoom, wouldn't it be great if you could just use Zoom to stream directly to Twitch and cutout the middle man?  Turns out you can!

## Setting up Zoom to stream to Twitch

Preamble aside, its fairly easy, but completely unintuitive to stream Zoom to Twitch. So lets talk it through. First of all you do need a paid [Pro, Business, Education, or Enterprise] account, hopefully you have that through your workplace.

Log into your organization's Zoom, and hit the **Settings** button on the left hand menu. Scroll down to **Advanced settings** and turn **Allow live streaming meetings** on. Check either the **YouTube** or the **Custom Live Streaming Service**, the latter which we'll configure for Twitch.

![Zoom Settings Page](/blog/streaming-from-zoom-to-twitch/zoom-settings.png)

Since I'm bad at remembering things, I throw the following in the instructions text box so I remember when setting up the meeting how to do it.

```
stream url: rtmp://live.twitch.tv/app
stream key: <your twitch account stream key>
live streaming url: https://twitch.tv/accountname
```

Then just hit Save. You're now ready to stream. You can either set up the actual stream live in a Zoom meeting, or you can Schedule a meeting and configure it there. Let's take a look at the latter.

## Schedule a Zoom meeting with Twitch Streaming

In your Zoom account on the zoom website click **Schedule a meeting**.

Set a name, the start time, duration etc and click **Save**.

This will drop you to the settings page for the meeting you just scheduled and on the bottom of that page is the **Live Streaming** configuration setting. Click **configure live stream settings**.

![schedule a meeting](/blog/streaming-from-zoom-to-twitch/schedule-a-meeting.png)

Here you enter the details following the instructions you conveniently placed in the Stream settings:

*You can find the twitch key at the following URL https://dashboard.twitch.tv/u/<username>/settings/channel*

![configure live stream](/blog/streaming-from-zoom-to-twitch/configure-live-stream.png)

Hit Save. Now when you join the Meeting it will automatically configure the Live Stream, but you'll still need to start the Stream by clicking the **... More** button and select the streaming option from there.

*If you didn't configure live streaming for the meeting, you can actually configure it here as long as you've enabled streaming in Settings*

![configure live stream in meeting](/blog/streaming-from-zoom-to-twitch/from-in-zoom-meeting.png)

Zoom will redirect you to a browser and configure the stream before redirecting it to your twitch streaming page.

![starting stream](/blog/streaming-from-zoom-to-twitch/start-stream.png)

You can close this twitch streaming window once its started or leave it open. If you leave it open turn off its sound or you'll create some fun echo loopbacks.

Here's the Twitch view:

![twitch stream](/blog/streaming-from-zoom-to-twitch/twitch-streaming.png)

Here's the Zoom View:

![zoom stream](/blog/streaming-from-zoom-to-twitch/zoom-streaming.png)
