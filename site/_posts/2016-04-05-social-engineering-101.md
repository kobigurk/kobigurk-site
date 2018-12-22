---
title: Social engineering 101
author: Kobi
---
Last Thursday we had our demo day in Boost.VC tribe 7. Excitement, adrenaline and expectations are high - the meetings we're going to have now, the connections that will assist us in building the companies we are dreaming of.

But there's another side to that. This can be exploited. John, a fellow tribe member, says that "people are our greatest resource". It's also true that they're our greatest weakness when it comes to security. 

People are usually easier to manipulate than machines, because we don't operate by a clear set of rules. We act more like a chaotic system, where one little detail can change entirely the way we behave. Especially when there are feelings involved.

It became especially clear to me when I noticed my co-founder sending an e-mail with our investor deck to a gmail account. Although in his case he was responsible, it could have been an entirely different situation. There's no problem creating a gmail account that resembles a real one, by collecting some details from a linkedin account, for example.

So I set out to prove this point. The tools I used:
1. Domain bought for a few dollars, with free e-mail by zohomail.
2. Social Engineering Toolkit.
3. Some data gathering from linkedin.
4. HTML5 template.

First, I built a plausible story. It's the first Monday since demo day, so it makes sense that an investor would follow up at this point of time. So I crafted the following e-mail:

>Hello X,

>I attended Boost VC's demo day and was impressed by your presentation.
We are looking invest in seed rounds of VR related companies.

>Could we schedule a chat for next week?
Meanwhile, I invite you to check out the details of the event we're hosting next week, it would be great if you could come: [Register](). 

>Best,

>Paul

I made sure that Paul had a background story. When you google his name, you find out he's an investor in VR companies. The e-mail also comes from a domain that makes sense. I could have done this part better, though, but more on this later.

With the Social Engineering Toolkit, I generated a page that looks exactly like g-mail's login page, but instead forwards the login details to my server of choice:
![login](/assets/images/se101-1.png){:style="width: 100%"}
This kind of makes sense, because you might need your gmail account to register to the event.
 
As it was just to prove a point, I didn't capture the login details. After recording the e-mail address, The user was forwarded to the user to the following image:
![hack](/assets/images/se101-2.png){:style="width: 100%"}


The results of this were that everybody I sent this e-mail to clicked the link, but no one actually filled in the g-mail account details. I believe this part was my fault. 
Some ways I could have done it better:

1. Instead of collecting e-mail accounts, I could have used a known exploit in a PDF or Chrome. It wouldn't work on everyone, but would be much more effective - everybody opened the link.

2. I could have also built the website for the company. One of the potential victims told me he did some research: he googled the guy, and that kind of made sense. But then he tried going to the VC website, which didn't exist. He still wouldn't put his details, because he "has been taught better", but he had a point. To remedy that, I downloaded a free HTML5 template and edited it so it would look like a new VC website (I think it looks better than some VC websites I've seen...):
![VC](/assets/images/se101-3.png){:style="width: 100%"}



To summarize, in less than an hour, it's possible to build a successful phishing attack - given that you can relate to the situation the other person is at.

Social engineering is dangerous. It happens all the time, and not only by computer hackers. We have to be wary of the e-mails we get, the links we click and, more generally, who we can trust.
