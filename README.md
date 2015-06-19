
THIS PROJECT IS CURRENTLY IN DEVELOPMENT. This is not all working yet. Check back later if you want a solution that has been tried and tested.

This is a write-up of a complete live-streaming-on-a-trolley system for use in e.g. hackerspaces that is easy to use, gets really good results while keeping costs low and using as much free and open source software as possible. 

The documented system has the following features:

* Full 1080p live-streaming
* No need to pay for a streaming service (but you can if you want)
* Use any camera with HDMI output
* Switch between camera output and output from presenter's laptop during streaming

This project was implemented by [Marc Juul](http://juul.io). Equipment purchases were funded by [Real Vegan Cheese](https://realvegancheese.org) as part of our [$5000 broadcasting stretch goal](https://www.indiegogo.com/projects/real-vegan-cheese). Thanks to the [sudo room](https://sudoroom.org) hackerspace for equipment donations and for help implementing and testing.

# Overview

Live streaming currently uses flash on the client and the RTMP protocol for content. This will change in the future to WebRTC. The only live output from decent (non-webcam) cameras is HDMI (more high end rigs use SDI). The two main challenges in live-streaming is:

* Converting the HDMI signal to a VP8/H.264 encoded network stream.
* Serving the network stream to multiple simultaneous clients.

We will need to deal with two simultaenous HDMI streams:

* One of from the camcorder capturing the scene (lecturer, meeting, event)
* A secondary stream for e.g. streaming the output from the presenter's computer

The secondary stream must accept all of the weird laptop output signals people use: HDMI, DVI, VGA, DisplayPort, Thunderbolt etc.

# Open vs. proprietary standards

Almost all web streaming today uses the Flash plugin on the client side. This means that your clients must use the proprietary and closed Flash player and that you are forced to encode your video as H.264 and AAC which are both patent-encumbered. The protocol used it RTMP which os also patent-encumbered.

The new WebM HTML video playback doesn't seem to have been designed for use with live streaming and the experiences of this author is that it does not work well.

WebRTC is the future of live streaming. It uses VP8 and Ogg Vorbis which are 100% free and open standards but it was first designed for one-to-one video-conferencing and the WebRTC server solutions (e.g. janus-gateway) don't seem entirely production-ready yet. Probably it will be ready for production use soon though!

All of this means that we are stuck with H.264 + AAC + RTMP + Flash until we the WebRTC software matures. This is do-able since open source implementations of everything we need is available. You might be breaking the law if you compile the code and redistribute the resulting binaries, but I'll leave that up to your judgment as I am not a lawyer.

# Encoding (HDMI to RTMP stream)

There are three basic ways of tackling encoding:

* Use a dedicated piece of hardware strapped to the camera 
* Use a camera with built-in encoding and RTMP output
* Use an HDMI input device hooked up to a normal computer (software encoding)

## Dedicated encoders 

These are all encoders that can take a HDMI input, encode it using H.264 and send an RTMP stream over ethernet to a user-specified RTMP server. Some or all of these may rely on cloud services to be configured (need to investigate).

* Teradek VidiU - $700
** Up to 1080p
* Cerevo LiveShell - $250
** SD quality only
* Cerevo LiveShell Pro - $700
** Up to 720p 

The following products are vendor-locked so you cannot use them with your own RTMP server:

* Livestream Broadcaster - $500
** Up to 720p

This document will not further discuss how to use this solution as it is pretty straight-forward and we opted to go for software encoding.

## Cameras with built-in encoders 

There are two options here: 

*The Raspberry Pi with the Raspberry Pi camera
*A camcorder with built in wifi streaming.

The Raspberry Pi is actually pretty amazing. Its built-in encoding + the fact that it runs linux means that it has amazing flexibility. On top of that it is super cheap. Unfortunately the only camera available doesn't have a very high quality sensor. It doesn't come with any decent optics, so you have to basically build a camera house and add your own optics, but the sensor is tiny so capturing enough light in anything but full daylight is near impossible (it will automatically reduce frame-rate when in low light) and good finding any optics that don't introduce an insance zoom factor.

The wifi camcorder option is a bit iffy, mostly because wifi on 2.4ghz at a conference-type event is likely to be bogged down, and because it's unclear if any of these camcorders allow you to stream to your own RTMP server (I doubt it). This solution also locks you into H.264 + RTMP.

## Software encoding 

Software encoding is by far the most flexible. It does not lock us into any one encoding or protocol. This is really useful since there are two changes coming in the near future:

* WebRTC is taking over from RTMP + Flash
* H.264 and VP8 are being replaced by the newer and better H.265 and VP9 codecs

With software encoding on a normal computer there are four concerns:

* Getting the HDMI output from the camera to the computer
* Getting the HDMI signal into a normal computer
* Having enough IO and CPU power to real-time encode H.264 + AAC content
* Turning the H.264 + AAC stream into an RTMP stream

### Getting the HDMI signal to the computer

The encoder server can be kept close to the camera but even if that is the case you will want the HDMI feed from the presenter's computer as well and for a lecture/performance hall of any size that means a running the HDMI signal longer than what is possible with a normal HDMI cable.

You could also have two encoders and then figure out how to switch in software, but that is more expensive and outside the scope of this guide.

The professional solution is to convert HDMI to the more professional SDI at the camera, but the converter is expensive:

*[Black Magic Mini Converter HDMI to SDI 4K - $300](https://www.blackmagicdesign.com/products/miniconverters)

SDI cable seems to be similarly priced to HDMI cable. 

You can also get active cables from RedMere that can run HDMI up to 60 feet:

* [60 feet on monoprice for $68](http://www.monoprice.com/Product?c_id=102&cp_id=10255&cs_id=1025507&p_id=9173&seq=1&format=2)

You can also get HDMI extenders that will send an HDMI signal up to about 300 feet using cheap CAT6 cable:

* [HDMI 300 feet extender for $90](http://www.conversionstechnology.com/HDMI-over-CAT5e-p/ct60s.htm)
* [HDMI 90 feet extender for $28](http://www.aliexpress.com/item/1080p-HD-Signal-HDMI-Extender-up-to-30-Meter-by-Cat5e-Cat6-Cables-W-Power-Adapter/32303950668.html)

### Capturing the HDMI on a normal computer

Black Magic Design has a few different devices for getting HDMI into a computer. One device is USB3 for windows and another is Thunderbolt for Mac. There are also other cheaper products like the [http://hauppauge.com/site/products/data_hdpvr2-gaming.html Hauppauge HD PVR 2] which also does encoding, but none of these devices are supported in Linux :(

The only device with HDMI input and linux support seems to be the [https://www.blackmagicdesign.com/products/decklink Black Magic Design DeckLink Mini Recorder] which is a PCI Express card. At $145 it is fairly affordable.

### Encoding content 

We'll use:

*To capture: [bmdtools](https://github.com/lu-zero/bmdtools)
*To encode and stream to server: [libav 10][https://libav.org/download.html] with x264 (H.264 encoder library) and faac (AAC encoder library)

We're using bmdtools to capture from the Camera. You'll need a copy of the Black Magic Design DeckLink SDK (sometimes referred to as the Desktop Recorder SDK) which contains the header files. You can download this from the Black Magic Design website (Their site sucks for this. Just go to their downloads page use your browser's built-in search function to find the SDK download link).

ToDo More documentation on how to set this up.

# Serving the RTMP stream

The server simply accepts an incoming RTMP stream and forwards to every client that connects. This can be run on a VPS.

## Hardware and bandwidth

A VPS with 100 mbit unlimited bandwidth (enough for ~80 simultaneous dvd-quality stream) is only [http://www.vpscheap.net/enterprise-vds.aspx $17 per month].

Compare to [https://www.ustream.tv/platform/plans $99 per month for 100 ad-free viewer hours].

## Server software 

* [RTMP streaming with nginx](https://obsproject.com/forum/resources/how-to-set-up-your-own-private-rtmp-server-using-nginx.50/)

The RTMPd server is also fairly easy to use, and Red5 is another open source option (but it's java based... yuck).

## Client software

You'll need a flash video player for the client side streaming. There are a few Open Source players:

* [f4player](https://github.com/gokercebeci/f4player)
* [flowplayer](https://flowplayer.org/) (DO NOT USE!)
* [mahara-flashplayer](https://github.com/catalyst/mahara-flashplayer) (Fork of flow player)

I really do not recommend using flow player. It's open source but they limit the free version to 4 minute videos and overlay their logo. You can edit the source code to remove these limitations and recompile it, but that's a huge hassle and they know it. Fuck 'em. Use a fork of flowplayer if f4player isn't good enough for you.

## RTMP script injection ==

Even though RTMP is non-open and therefore sucks, an interesting feature is that it can contain arbitrary actionscript objects that are time-synchronized and trigger javascript function calls in the user's web browser, providing the ability for slide change events, chat transcripts or annotations to be saved as part of the stream and replayed. 

We're not currently using this ability, but for saving the stream for later playback with e.g. a time-synchronized chat this would be really useful. We have some prototype code [in libcuecumber](https://github.com/UniversalPrimer/libcuecumber) that could be used to do this (thanks [hdweiss](https://github.com/hdweiss)!).

# Audio 

ToDo write me

## Getting audio from the presenter

On our main stage we're running everything from a central mixer/amplifier so it's simply a matter of getting the stereo output from out mixer to the streaming server. This can be done easily enough by running a separate stereo cable (another shielded CAT5/5e/6 cable works fine) along with the CAT6 carrying the HDMI signal (I recommend taping them together at several points). 

If you are OK with only having mono audio from the stage then you might be able to use the following hack: Get an HDMI over CAT6 extender system that supports also extending an Infrared signal and use this infrared port to send the audio from the stage. If you are only recording from a microphone input then this works great. If you are recording from a laptop output as well then you will have to convert stereo to mono (this is easy, requiring only three resistors and a bit of soldering).

We have a wireless lavalier microphone included on the live streaming cart for situations where we're not near the main stage. Here we have the microphone receiver

* An old Samson TX-3 wireless lavalier mic (we got it donated)
* An old Samson MR-1 wireless receiver (also donated)
* A [Behringer MicroMIX MIX400 4-channel mixer](https://www.zzounds.com/item--BEHMX400) for $25

Ideally we'd also have a hand-held wireless microphone that sends on the same frequency that can be handed around when there are multiple people / audience questions. We might invest in a multi-mic system at some point. The important part is that the receiver and mixer is mounted on the cart and you can easily adjust input levels during recording. Four channels lets us have individual control over three microphones and the output from the presenter's laptop.

I recommend getting the wireless mic equipment used. The cheap stuff can be unreliable and the good stuff is expensive if you get it new. Honestly, sound quality is more important than video quality for most things people want to live stream so don't skimp on the sound!

# The server hardware

We're using one of those very compact desktop computers: A HP Compaq DC7700. It's small, fairly low noise and fairly low power. An even lower noise and power mini-itx type solution would be more ideal, but we wound this one in a free-pile on the street (broken harddrive) so we decided to use it. We had to physically hack open the case to make room for the slightly-too-tall HDMI input card. Anything with a Core 2 Duo or better processor and 2+ gigs of ram should be enough.

# The camcorder

It might be tempting to use a nice SLR but BE CAREFUL! They do not have active cooling and may overheat if you're using them for a long time (hours). If you want to try, you'll need something that runs the Magic Lantern firmware since the stock firmware on SLR cameras won't even let you run for that long.

We're using a Sony HDR-PJ540 since we also use the same camera for amateurs to film their projects and the Balanced Optical SteadyShot really smooths out even the rough motions of a hand-held camera in untrained hands. It works fine for recording presentations as well.

# The USB sound card

The built-in sound cards in many computers are prone to noise. 

ToDo select a good USB sound card.

# Putting it all together

We put everything on a nice wheeled raise-lower'able trolley that we got for free off Craiglist. No idea where to get a similar one, but really any trolley will do.

Here's what we have on the trolley:

* The encoding server with the HDMI input card, a wifi card and a USB sound card
* A UPS so someone accidentally unplugging the power won't ruin the show
* An LCD monitor, a mouse and a keyboard for the server
* An HDMI to HDMI + Analog audio box (so the audio can be piped through the mixer)
* A wireless microphone receiver
* A 4-channel audio mixer
* An HDMI over CAT6 extender receiver box
* An HDMI two-port input switcher box (to switch between camera and presenter's laptop)
* A pair of headphones for the person controlling the camera and mixer
* A long 110v AC power cord to power the whole setup
* A box with the camcorder we use
* A nice smooth tripod for the camcorder
* A Monoprice RedMere HDMI cable for camcorder (normal HDMI cables are too thick and unwieldy)
* A box of stuff for use on the stage
* A box of adapters and converters from every modern video plug to HDMI
* Whichever cables are needed to connect everything on the trolley

A lot of the stuff on the trolley is held down with velcro. Tip: Use self-adhesive velcro but if attaching to metal, heat both the glue on the velcro and the attachment surface a bit with a hair dryer or heatgun (don't overdo it and ruin electronics!) just before sticking it together. It will stick _much_ better and hold for longer. 

The box of stuff for use on the stage contains:

* An HDMI splitter (so presenter's laptop output can go to both projector and encoding server)
* An HDMI over CAT6 extender sender box
* A long CAT6 cable taped to a long shielded stereo audio cable
* A lavalier wireless microphone and transmitter

The box of adapters we have contains:

* A mini-HDMI to HDMI adapter
* A DVI to HDMI adapter
* A Mac Thunderbolt or DisplayPort (these two are compatible) to HDMI adapter
* A mini-DVI to HDMI adapter (some older MacBooks use them)
* A PC DisplayPort (these are very different from Mac DisplayPort. Some PC laptops have them) to HDMI adapter
* A VGA to HDMI converter box (an adapter cable is not enough since those generally only work for Mac laptops)
* A Chromecast device + AC to USB power adapter (for android devices)
* An Apple TV (for iOS devices)

I recommend buying the adapters, HDMI splitter, HDMI extender, etc. on ebay or deal extreme. Get the cables from Monoprice.

# Planned improvements

## Improved camera HDMI connection

HDMI ports are not the best for a production environment. It is easy to accidentally unplug the cables or pull them sideways in a way that ruins the plug or port. The RedMere HDMI cables from Monoprice help a lot since the thinner and more flexible cables put less strain on the connectors. 

In the future we will likely use a re-purposed DLSR cage (find them for less than $100 on ebay, deal extreme or ali express) and some moldable plastic like InstaMorph with embedded nuts. We'll wrap the instamorph around the male HDMI connector and embed a two or three stacked nuts, such that a bolt can connect the cage to the nuts on the HDMI connector, holding the connector in place.

## Modding the UPS

We plan to modify the UPS so it blinks lights at the camera-person instead of emitting a disruptive loud noise when it's unplugged.

## Better wifi

We're planning to upgrade the wifi. Currently we're using an old 802.11g PCI card. We'll likely use a pair of TP-Link N750 routers with WPA2 security on a dedicated 5 GHz channel. Either that, or use a pair of Ubiquiti Nanostation M900 routers to make a 900 mhz dedicated wifi network which is even less likely to become encumbered when lots of people are using the wifi. You could also use a 5 GHz PCI Express card, but that would require a computer with two PCI Express slots (ours has one) or a USB 5 GHz wifi adapter (more CPU usage + it is not clear if any USB2 wifi cards have Linux drivers). The cost for the 5 GHz solution is about $90 and the cost of the 900 MHz solution is about $260. The 900 MHz solution is much better if you plan for the signal to pass through multiple walls or lots of people or foliage.
