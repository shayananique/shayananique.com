---
title: 'MPNS module for Node: A simple push notification helper for the cloud'
date: 2011-12-28T08:32:13+00:00
categories:
  - Windows Phone
---
One of the fun projects I’ve been working on recently related to my app has been bringing online a cloud to handle sending push notifications processing. I’m using [Node.js](http://nodejs.org/) for this and have a ton of people enjoying the beta push experience right now: toasts, live tiles, etc.

Tonight I pushed to GitHub ‘mpns’, a simple interface and helper to the Microsoft Push Notification Service (MPNS). It essentially takes the simple properties for your live tile update or toast, packages it in a simple XML payload, and then posts it to the subscription endpoint.

It isn’t a lot of code, but I sure hope it helps others who may be experimenting with other platforms while building a great push experience. If you’re using Azure, the Windows Phone team has already provided some awesome content here – [Yochay has previously posted](http://windowsteamblog.com/windows_phone/b/wpdev/archive/2011/01/14/windows-push-notification-server-side-helper-library.aspx) about a [Windows Push Notification Server Side Helper Library](http://create.msdn.com/en-us/education/catalog/article/pnhelp-wp7).