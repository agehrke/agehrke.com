---
title: "Disable Sitecore filters for Web API controllers"
date: "2016-02-08"
categories: 
  - "dotnet-dev"
tags: 
  - "sitecore"
---

I have previously written about how easy it to implement [Web API controllers in Sitecore](http://www.agehrke.com/2015/08/web-api-controller-in-sitecore-8/ "Web API controller with async/await in Sitecore 8"). However since Sitecore adds a lot of global filters for authorization and security policies - e.g. see `sitecore/api/configuration/filters` section in `Sitecore.Services.Client.config`. You might want to disable these filters, as they can interfere when building a public accessible API - see gist below. During a debugging session I noticed that a filter from some FXM assembly were also added.

To be fair, most of Sitecore's implementations of `IFilter/ActionFilterAttribute` that are assigned globally on `HttpConfiguration`, does check for presence of `ServicesControllerAttribute` and inheritance of `ServicesApiController`. This means you might prevent these filters from doing anything by using `System.Web.Http.ApiController` as base class for your controllers. I haven't been though the code for all Sitecore's filters, so I'm playing it safe, and decorate my ApiControllers with a `ClearSitecoreWebApiConfigAttribute`.

<script src="https://gist.github.com/agehrke/8474a321e804fd0af1a0.js"></script>

I encourage the developers of Sitecore to consider using [controller-specific configuration](http://www.asp.net/web-api/overview/advanced/configuring-aspnet-web-api#percontrollerconfig) instead of loading the global `HttpConfiguration` with all kinds of module or feature specific stuff.
