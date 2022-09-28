---
title: "Web API controller with async/await in Sitecore 8"
date: "2015-08-05"
categories: 
  - "dotnet-dev"
---

So you want to create Web API controllers in a Sitecore 8 solution? Use `Sitecore.Services.Infrastructure.Web.Http.ServicesApiController` and add `Sitecore.Services.Core.ServicesControllerAttribute` to namespace your controller. Read [Developer's Guide to Sitecore.Services.Client (PDF)](https://sdn.sitecore.net/upload/sitecore7/75/developer's_guide_to_sitecore.services.client_sc75-usletter.pdf) from page 61 and onward. Controllers will be reachable at:

`/sitecore/api/ssc/{namespace}/{controller}/{id}/{action}`

That's it. No routing setup, no pipeline fiddling. Quick and easy! Just note that Sitecore does not support dependency injection in the controller - at least out of the box.

Below is a full example that shows async/await in a Web API controller - note the `aspnet:UseTaskFriendlySynchronizationContext` appSetting in Web.config.

You might also want to check my post on [disabling of Sitecore Web API filters](http://www.agehrke.com/2016/02/disable-sitecore-filters-web-api-controllers/ "Disable Sitecore filters for Web API controllers").

<script src="https://gist.github.com/agehrke/a3b3da83bfd9d7b109f9.js"></script>
