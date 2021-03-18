---
date: "2021-03-15"
title: "Streaming DevOpsDays Texas"
categories: ["twitch","youtube","streaming","OBS","devops","developer advocate", "dod"]
draft: true
---

I recently had the privilege to help organize DevOpsDays Texas 2021 a virtual event that we ran to try and scratch our DevOpsDays itch given that in-person conferences won't be happening any time soon.

I had the misfortune of being the most knowledgable about live streaming and therefore was nominated as the person to figure out how to live stream the event to Youtube.

I chose to use [Open Broadcaster Software (OBS)](https://obsproject.com/) to manage the stream and Windows as the OS to run it on (OBS support is generally best on windows).

## Setting up the streaming box

I already have a pretty decent streaming box at home, its a couple year old Dell workstation that I added an SSD and a Nvidia GTX 1050 ti. The most important component of an OBS machine is that it contains something capable of doing h.264 encoding in hardware, otherwise the CPU will be blasted.

The 1050ti is a few years old, but still supports `nvenc` which is the Nvidia library for doing hardware encoding. One important thing to note is that for `nvenc` to work you must have a display hooked up to the card. This means you cannot use the Windows Remote Desktop tool as it swaps out the display for a virtual one.

I use NoMachine as a remote desktop tool to connect to my streaming box and I have a [cheap HDMI dummy monitor](https://www.amazon.com/Headless-Display-Emulator-Headless-1920x1080-Generation/dp/B06XT1Z9TF/) which tricks the GPU into thinking it has a monitor plugged in.

I then installed [OBS Studio 26.1.1](https://obsproject.com/download) and two OBS plugins [Audio Monitor](https://obsproject.com/forum/resources/audio-monitor.1186/) and [Advanced Scene Switcher](https://obsproject.com/forum/resources/advanced-scene-switcher.395/) which I'll detail later.

Knowing that I would need to do some advanced audio work I also downloaded [VB-Cable Virtual Audio Device](https://vb-audio.com/Cable/) to use to ensure I could get audio from Zoom to OBS. For previous streams I've needed more cables and have used their A+B and C+D cable packs as well as [Voicemeeter Banana](https://vb-audio.com/Voicemeeter/banana.htm) for mixing the audio, but for this event I was able to get by with just the one Cable.

I knew I would want to run a backup streaming server in the cloud so I kept all my files in a sensible path `c:\obs\dod-tx` which meant I could export profiles and scene collections, copy up the whole thing and import them from the same location on the streaming server.

[IBM Cloud](https://ibm.com/cloud) kindly offered to provide us with a dedicated GPU server which we gladly took them up on. Unfortunately they did not have a Dummy HDMI monitor dongle so I was unable to take advantage of the GPU, but the CPU onboard was powerful enough to handle the encoding.

Knowing that I would need to keep files in sync between the two servers I looked into the state of the art for copying files between windows servers securely over the internet. To my chagrin I found no great answer and ended up doing something that I haven't done in fifteen or more years, installing [Cygwin](https://www.cygwin.com/) and rsync. I'm sure there is something better out there, but at least this gave me the ability to sync `c:\obs\dod-tx` between the two machines over SSH which is secure enough for me.

## OBS Plugins

### Audio Monitor

[Audio Monitor](https://obsproject.com/forum/resources/audio-monitor.1186/) is a plugin that allows you to apply one or more **Audio Monitor** plugins to any audio feed in OBS. For each audio source we set up two Audio Monitors, one to a set of headphones, the other to the VB Virtual Audio Cable which we could route to the Microphone source in Zoom and OBS.Ninja.

### Advanced Scene Switcher
[Advanced Scene Switcher](https://obsproject.com/forum/resources/advanced-scene-switcher.395/) is a plugin that lets you automate OBS changing scenes based on a number of triggers such as the current time, or when a media source finishes playing.

## OBS Overlay and Underlay Scenes

In OBS you create a Scene that contains one or more sources that are layered on top of eachother. You can use another Scene as a source. This means that you can compose fairly complex scenes together into the stream.

This allowed us to have a few master scenes that were the base for the rest of the scenes.

We had a basic Interstitial (that's a big word, I think I'm using it right) scene that consisted of a basic background which we could add into a specific scene with a text source over the top of it. This was used for breaks to show what was coming up next etc.

![interstitial scene showing kickoff time](./interstitial.png)
_This shows our starting soon scene which contains the interstitial scene and a text source that says "stay tuned we'll kick off at 9am"._

## The Conference Schedule

Obviously to build out DOD TX scene collections I needed to first determine the schedule. Thankfully we'd had a very successful CFP and we had two Keynotes, eleven 30-minute Sessions, two panels (which we called fire starter chats), thirteen 5-minute Ignites, and twelve sponsor pitches. We also had opening and closing messages for each day, speaker introductions, and a couple of how-to videos scheduled during breaks etc.

Most of the content was pre-recorded and we asked Speakers, Ignites, and Sponsors to provide 1920x1080 resolution videos. Quite a number of the speakers used Zoom or similar to do their recordings and we ended up with a lot of videos that were _not quite_ 1080p. This is relatively easy to fix with OBS, so it wasn't a big deal.

We did however want to do the closing  messages live each day, and the fire-starter chats were live, so we had two all day Zoom meetings that we could capture the audio/video from for those. Much to our annoyance, Zoom does require at least two people in a meeting at all times or it will eventually time the meeting out. This happened to us a few times, however we could just re-open the same meeting so it wasn't a big deal.

## Captions and Live Drawing

We knew that we wanted to ensure the conference was accessible so we opted to hire [White Coat Captioning](https://whitecoatcaptioning.com/) to do live captioning and we also hired Ashton from [Minds Eye Creative](https://www.mindseyecreative.ca/) to do a live drawing of the contents of each 30-minute session.

To do this I needed a way to get our conference audio/video live to both parties, Streaming services usually add some delay for processing to a stream, so it was important that we were able to provide them with an undelayed live video/audio feed. To do this I set up an [obs.ninja](obs.ninja) room.

In OBS Ninja I shared the OBS virtual camera and VB Virtual audio cable in the Control room, both the captioning team and Ashton would connect to this able be able to see and hear the live feed from OBS without any delays.

### Live Drawing

When Connected Ashton would see and hear the OBS feed live, and would share her camera/desktop via a custom receive URL.

![Ashton's view of the conference](./ashton-view.png)

_Ashton's view of the conference, when a session is running she sees the live video from OBS which includes the session video as well as her own screenshare, so she has a preview of what the live stream will see._

I then created a Scene in OBS called `_ashton - drawing` which contained two sources, and Image that would be displayed if Ashton's feed was offline and Ashton's OBS.ninja feed to which she shared her screen focussed on her drawing app.

![OBS Scene for Ashton](./ashton-scene.png)

_The OBS Scene containing Ashton's cartoon.  I've added a touch of transparency to her source so that you can faintly see the image behind it that would show if she was offline. During the conference this was totally opaque._

### Captions

The captioner would see exactly the same feed as Ashton above. However we didn't want to capture anything directly from White Coat, instead they send a text stream to a [streaming text](https://www.streamtext.net) website. The website would show an auto-scrolling wall of text as the captioners did their thing. I created an OBS Scene that contained a Browser source pointing at the text stream like so.

![OBS Scene for Captioning](./caption-scene.png)
_The captions scene shows the whole stream text website._

The captioners would blast carriage returns at the start of each day to ensure that when the captioning started it would be on the last two lines and the auto-scrolling would keep it there. This means we can add the `_captioning` scene above as a source into any scenes that we wished to display the captions and crop the source to show just the last two lines of the source.

![Captioning cropped out in session](./captions-in-session.png)

_The captions thus appear in a captions bar across the top of any scenes that we wanted captioned._
