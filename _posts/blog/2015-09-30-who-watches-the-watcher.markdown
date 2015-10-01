---
layout: post
title: "Who Monitors the Monitor?"
modified:
categories: blog
excerpt: The reason my last month as an intern turned me into an insomniac 
tags: []
image:
  feature:
date: 2015-09-30T22:25:12-07:00
---

I stated one of my early Co-op internships as a software developer for company that developed their product on top of an embedded Linux platform. Some of the company's features ran as a Linux process. Of course, if this process ever were to go down, we needed some way for the system to be able to restart the process.

When I arrived, there was a cronjob in place. Every minute, it would run a shell script that checked if the process was still working. If it wasn't, it would restart it. It was clunky, but it worked. However, our software lead discovered something cleaner and better and more versatile.

Enter [Monit](https://mmonit.com/monit/).

<figure>
	<img src="https://upload.wikimedia.org/wikipedia/en/b/b5/Tildeslash_Monit_Logo.gif" alt="image">
	<figcaption> Source: Wikimedia: Tildeslash Monit Logo</figcaption>
</figure>

Monit is fantastic. It's a background process that can be configured to do, well, monitoring. We replaced the cronjob's script with a simple monit configuration:

{% highlight sh %}
check process VITAL_COMPANY_PROCESS matching "vital_company_process"
	start program = "/etc/init.d/vital_company_process start"
	stop program = "/etc/init.d/vital_company_process stop"

{% endhighlight %}

Super. It would regex match our process name with all running processes and restart the program if it ever stopped. 

We also took this one step further for monitoring our web server. The configuration was something like:

{% highlight sh %}
check process WEB_SERVER matching "web_server"
	start program = "/etc/init.d/web_server start"
	stop program = "/etc/init.d/web_server stop"
	if failed 
		port 80
		protocol http
		request "/watchdog_page.html"
		and content = "woof woof still alive"
		then restart
{% endhighlight %}

Monit is a champ.

---

Fast forward 5 months. It's nearly the end of my Co-op internship. I have 2 months to go before my contract ends. I'm also in charge of developing a critical feature that the company is about to ship the same week I was to leave.

Fortunately, development went smoothly. Design had been done for about a month before the implementation was assigned to me. I finished coding in about 3 weeks. The unit tests I made all passed. Integration tests were all green. Valgrind gave me the glorious ***All heap blocks were freed -- no leaks are possible***. We were ahead of the schedule, giving me ample time to comfortably test and verify. I had five weeks before ship day. 

Finally, it was time to do long-term stability testing. I set up a system to run overnight and went home, hopeful.

I arrived in the office the next day and noticed in the logs that the VITAL_COMPANY_PROCESS had been restarted. I opened Monit's logs. Monit noticed the process had stopped running and restarted it. Thanks Monit.

Oddly though, the system logs showed no reason why the process stopped. It was a silent crash, it seemed. SegFault perhaps? I attached GDB to the process, turned off Monit, and left the system overnight to see if I could catch it the next day stopped in its crashed state.

The next day, the program never crashed though. Weird. Must have been a glitch. We still had lots of time to test, so I started testing more systems overnight.

Again, I would come into the office in the morning and most systems would have crashed. We never noticed this issue before. There must have been something in my feature that created this issue.

The next two weeks went by blazingly fast. Each day I would meet with my dev lead. We hypothesized which part of the feature could cause system-wide issues. We refactored tremendous parts of the new code. We dumbed down the feature and took smaller components out. 

Every day, we would encounter the silent crash. GDB wouldn't be able to return a core dump; it only would report ***No stack.***. The crashes would occur intermittently at random time intervals. Some processes would restart in 45 minutes, others after a day and a half. We had nothing to work with.

We eventually took the entire feature out. This changed nothing. Our system still would not go for a long time without the process silently crashing and restarting. At this point we realized it couldn't be my new feature causing the issue. Something is happening to the system, it's been happening for a long time, and we never noticed until now.

---

Finally, it was the last weekend before ship day. A bunch of higher-ups and the entire development team sat down with me in a meeting. We needed to fix stability. We needed to ship.

We set up as many test units as we could. We wrote our own C code to catch **SIGSEGV/SIGBUS/etc** and manually generate a core dump. We turned on GDB and turned off Monit. We only had this one weekend to find the issue, or else we were screwed. It was Friday 7pm. The entire team was still in the office.

---

My team and I mentioned in jest that maybe it was Monit that was killing our programs.

---

It wasn't until the next day we started considering this seriously. Many of us went in on Saturday to check the systems. We had zero crashes after about 12 hours of uptime. This was uncanny.

The same was on Sunday. Zero process restarts after over 36 hours of uptime.

I started doing some digging into Monit. Monday we had zero crashes after 60 hours. And I had an answer.

---

Turns out it was Monit all along. We were using an older version of Monit that would "restart" a process it didn't detect by calling STOP, then calling START.

And it turns out our **/etc/init.d/vital_company_process stop** script just sent SIGKILL to our process. Which explains the silent crash.

And forums for Monit were saying how process checking with regex should be avoided whenever possible in lieu of process checking by PID. People were experiencing false negatives with Monit when it was checking via regex.

---

In hindsight, it was dumb that we didn't suspect the monitor program we were using would be at fault. We made the easy fixes and verified our system to be shippable. We continued vigorous long-term testing after shipping to ensure that the system continued to be stable after long amounts of uptime.

My last month with the company was definitely a sleep deprived one. What appeared to have been a clear path to a ship date became a frantic month of code witch-hunting. 

Lesson learned. Never let your guard down, and never trust any code. Even the monitor.

---

*It should be noted that we were using an older version of Monit. Monit's behaviours that caused issues for us were fixed in more recent versions. I still believe that Monit is a fantastic program. Shout out to Tildeslash for his hard work! You can check out Monit's [official site here](https://mmonit.com/monit/) or the [code repository here](https://bitbucket.org/tildeslash/monit).*
