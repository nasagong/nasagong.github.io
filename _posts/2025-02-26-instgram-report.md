---
title: My IG bug bounty experience
description: experience about IG bug bounty
author: nasagong
date: 2025-02-26 11:10:00 +0800
categories: [Misc]
tags: [Web]
render_with_liquid: false
---

> No paid content has been exposed in this article. 

> This post is purely based on technical curiosity, not intended to encourage the piracy of paid content.<br/>


## 0. Intro

![](/assets/img/2025-02-26-16-06-02.png)

I use Instagram pretty often, but I mostly stick to posting Stories rather than directly uploading to the feed.  
If you’ve ever tried using Instagram on PC, you probably know this already — grabbing the original files for Stories or Feed posts is ridiculously easy.  
In fact, there are tons of third-party apps out there that let you download media from other accounts.

Just from the screenshot above, you can see that you can simply open DevTools and check the URL path of the original image.

## 1. Instagram’s Paid Subscription System

![](/assets/img/2025-02-26-16-09-13.png)

Fairly recently, Instagram rolled out a paid subscription system.  
With so many different subscription platforms popping up these days, I guess Instagram wanted a piece of the pie too.  
While using Instagram on PC, I suddenly got curious —  
is it just as easy to grab the original URLs for paid subscription content as it is for regular media?

![](/assets/img/2025-02-26-16-18-52.png)

Turns out, on mobile, if you try to screenshot or screen-record paid media,  
there’s a Netflix-style capture protection system in place that just replaces everything with a black screen.  
(Sorry, can’t share actual capture samples here due to copyright reasons.)  
But then, what about on PC?  
I found a legit paid subscription account and ran some tests on a desktop environment.


### In the Case of Images

![](/assets/img/2025-02-26-16-24-49.png)

For images, just like with public Stories, I was able to access the original file by hitting the generated CDN URL.  
Even when I sent a raw HTTP request without any token information attached,  
the server still responded with the original image without any problems.

### In the Case of Videos

![](/assets/img/2025-02-26-16-26-54.png)

As for videos —  
a bit of Googling showed that up until around 2022, the original video sources were directly exposed just like images.  
But when I checked now, that method no longer worked.  
I figured they might be streaming the video as chunks instead,  
so I opened DevTools, switched the network filter to Fetch/XHR, and sure enough,  
I could see multiple video chunks being fetched.

After sorting the requests by size and checking the Request URL of the biggest one,  
it turned out they were indeed coming from a CDN server.

Of course, since the videos are streamed,  
just firing a simple GET request at that URL gives you an empty or broken video like this:

![](/assets/img/2025-02-26-16-30-32.png)

The URL was super messy, so I used a [URL parser](https://www.freeformatter.com/url-parser-query-string-splitter.html) to make it easier to read.

![](/assets/img/2025-02-26-16-32-37.png)

Digging into the query string, I found some very promising parameters: `bytestart` and `byteend`.  
As expected, deleting those two parameters gave me access to the full original video for download.

Just like with images, even in a sandbox environment,  
sending a raw HTTP request to the URL successfully returned the full media.


## 2. Question

It was pretty obvious that this setup was intentional by Instagram.  
It’s just not realistic to think that Meta would miss something this simple in their design...  
Still, if getting the original media for paid content is this easy,  
it feels like it opens the door for non-subscribers to leak content,  
which could end up partially violating creators' rights.  
Especially since I had already confirmed that on mobile, even basic screenshots are blocked — it made me even more curious.

At first, I assumed that accessing the generated URL would require some kind of token tied to a logged-in session.  
But being able to pull it off so easily without logging in felt strange.

Of course, it’s not like I could easily reach out to Meta’s security team directly.  
So while digging around with Google and AI tools on my own,  
I eventually thought of bug bounty programs.

Obviously, I knew this wouldn't really qualify as a valid security bug...  
But still, for the sake of satisfying my own curiosity — even if it might be a bit annoying for their security team —  
I decided to go ahead and try writing up a report.

When I asked a few friends working in security about it,  
they basically told me, “Eh, just send it and see what happens,”  
so I decided to go for it.


## 3. Reporting

![](/assets/img/2025-02-26-16-42-23.png)

As you’d expect from a big tech company, Meta has a flashy official bug bounty page.  
I clicked the report submission button, wrote up a simple PoC, and sent in my report.

## 4. Result

Maybe because it was such a minor issue, I got a response way faster than I expected.

![](/assets/img/2025-02-26-16-47-28.png)

Long story short:  
they told me this was intended behavior,  
that it’s not something they can (or are trying to) block,  
and that since only users with permission can generate those URLs, it’s not a security problem.

Sure, once a user has access, you could try to add technical barriers to stop them from saving the file,  
but realistically, if someone really wants the file, they'll always find a way to get it anyway.  
Still, the way they responded so casually and openly was kind of surprising.

Thinking about it more —  
even if they had beefed up CDN security like I initially imagined,  
it probably would’ve just worsened the user experience and massively increased maintenance costs.

From this experience, I realized that in service security design,  
**perfect protection is basically impossible** and **making efficient trade-offs matters way more**.

At first, I thought it was bad security practice that the CDN served paid files without extra verification.  
But after realizing it was an intentional design choice, it made a lot more sense.

When you’re running a service like this,  
you have to balance security strength, user experience, and cost efficiency.

For example, if you really wanted to tighten security,  
you could force OAuth tokens for every CDN request or add IP-based session control.  
But honestly, with Instagram’s scale,  
the overhead per request would skyrocket,  
and the traffic would be on a whole different level — not really practical.

![](/assets/img/2025-02-26-16-58-37.png)

It would also kill a lot of the benefits of CDN caching,  
and the overall UX would get noticeably worse.

At the end of the day, Instagram isn’t trying to be a private file vault like Dropbox or something.  
Their whole model relies on people actively sharing media to drive engagement and revenue.  
From that perspective, it actually makes perfect sense that they don’t go overboard with protection.


### Then Why Is Mobile Protected?

Following the previous logic,  
it seems like the reason they apply protection on mobile is simply because it's way cheaper to do so.  
After checking a bit more, it turns out that unlike PC environments,  
applying DRM at the OS level on mobile is a lot easier.

I couldn’t find a great source to link here,  
but if you're curious, you can Google around with keywords like  
**FLAG_SECURE**, **ScreenCaptureProtection**, or **Hardware-based DRM**.

### Closing Thoughts

Until now, I’d only ever looked at security from a distance,  
and I was always curious about how actual security architecture gets designed.  
Through this experience — even if it was shallow —  
I got a glimpse into how real practitioners think, and it was honestly pretty valuable.

That said...  
to save security teams some headaches,  
I also made a little promise to myself to handle my personal curiosity on my own from now on...

## References


[How does netflix DRM work?](https://www.reddit.com/r/webdev/comments/3p8bos/how_does_netflix_drm_work/)