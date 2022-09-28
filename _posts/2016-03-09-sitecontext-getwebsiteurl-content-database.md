---
title: "SiteContext.GetWebSiteUrl() - Hidden use of ContentDatabase"
date: "2016-03-09"
categories: 
  - "dotnet-dev"
tags: 
  - "sitecore"
---

I was recently looking at [Sitecore composite components](http://www.chrisvandesteeg.nl/2015/11/03/sitecore-composite-components/) by [Chris van de Steeg](http://twitter.com/csteeg). It's a great and useful concept which he explains well. I grabbed his implementation on Github to give it a try.

Unfortunately the code in [CompositeComponentController](https://github.com/csteeg/BoC.Sitecore.CompositeComponents/blob/master/src/Controllers/CompositeComponentController.cs) threw a NullReferenceException inside a call to `SiteContext.GetWebSiteUrl()`. Having never used that particular method before, I turned to my primary Sitecore documentation tool - the decompiler. Turns out the method calls an overloaded version of it self by passing a `Database` as argument - but not just any database, `Sitecore.Context.ContentDatabase`.

Maybe I'm too much of a Sitecore newbie for asking this, but what is the content database in code running on a CD-server? In my solution it's `null`. Decompiling into the `ContentDatabase` property of `Sitecore.Context`, I see that you can set a content database on the site definition config using a `content` attribute. Should I set it to _web_ on CD-servers?

Have you experienced other Sitecore APIs that make hidden use of ContentDatabase?

[![Use of ContentDatabase in SiteContext.GetWebSiteUrl](images/SiteContext.GetWebSiteUrl.png)](http://www.agehrke.com/wp-content/uploads/2016/03/SiteContext.GetWebSiteUrl.png)
