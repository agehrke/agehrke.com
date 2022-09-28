---
title: "Proposal for Sitecore.Services.Client and Web API in Sitecore"
date: "2016-04-28"
categories: 
  - "dotnet-dev"
tags: 
  - "sitecore"
---

In this post I'm going to dive into how the infrastructure for Sitecore.Services.Client is implemented and how it overrides many defaults in ASP.NET Web API such as filters, MediaTypeFormatter and controller selection. Lastly I'm going to look at how Sitecore can improve their usage of Web API. **Disclaimer:** All information are gathered by myself by investigating Sitecore 8.1 Update-2 configuration files and assemblies, which may mean that's not completely accurate.

### What is Sitecore.Services.Client

> Sitecore.Services.Client provides a service layer on both the server and the client side of Sitecore applications that you use to develop data-driven applications. Sitecore.Services.Client is configurable and extendable, and the framework and the "scaffolding" it gives you help you create the client-server communication in an application in a consistent way. Sitecore.Services.Client uses the ASP.NET Web API as a foundation

This is Sitecore's own description of Sitecore.Services.Client - taken from [Developer's Guide to Sitecore.Services.Client (PDF)](https://sdn.sitecore.net/upload/sitecore7/75/developer's_guide_to_sitecore.services.client_sc75-usletter.pdf).

## Details of infrastructure

Sitecore.Services.Client are setup and configured by use of an `initialize` pipeline processor - defined in Sitecore.Services.Client.config. The processor `ServicesWebApiInitializer` is very thin, and instead delegates a lot of the configuration to `ServicesHttpConfiguration`. What basically happens is:

1. Enable CORS
2. Replace `IHttpControllerSelector` service with `NamespaceHttpControllerSelector`
3. Map Web API routes defined in `DefaultRouteMapper`
4. Add `IFilters` defined in `api/services/configuration/filters` configuration to `HttpConfiguration.Filters`
5. Add Sitecore's BrowserJsonFormatter to `HttpConfiguration.Formatters`

### 1: Enable CORS

Absolutely nothing special here. Sitecore just calls [EnableCors() extension method](https://msdn.microsoft.com/en-us/library/system.web.http.corshttpconfigurationextensions.enablecors(v=vs.118).aspx) on HttpConfiguration.

### 2: Replacement of IHttpControllerSelector with NamespaceHttpControllerSelector

Sitecore changes the controller selector with their own class by _globally_ replacing `IHttpControllerSelector` service on `HttpConfiguration.Services`. This affects all Web API controllers in entire solution, which can break things in some not so obvious ways for the user/developer. The replacing class extends the default controller selector, appropriately named [`DefaultHttpControllerSelector`](https://github.com/ASP-NET-MVC/aspnetwebstack/blob/master/src/System.Web.Http/Dispatcher/DefaultHttpControllerSelector.cs), by implementing grouping or namespacing of controllers based on a `NamespaceQualifiedUniqueNameGenerator` which reads namespace from controllers decorated with `ServicesControllerAttribute`. This is somewhat like areas for ASP.NET MVC. It works great - most of the time. After upgrading to Sitecore 8.1 all Web API controllers started to return _"An error has occurred"_ on CD-servers. As you can read in my [blog post about the subject](http://www.agehrke.com/2016/04/web-api-error-sitecore81-cd-servers/), the root cause was related to `NamespaceHttpControllerSelector`.

### 3: Mapping of routes using DefaultRouteMapper

The `DefaultRouteMapper` class registers a bunch of Web API routes using [`MapHttpRoute()`](https://msdn.microsoft.com/en-us/library/system.web.http.httproutecollectionextensions.maphttproute(v=vs.118).aspx) - nothing special there. All routes are defined with a prefix, which defaults to _sitecore/api/ssc_. Some routes define controller name and optionally action name. But a wildcard route is also defined as `{namespace}/{controller}/{id}/{action}`. This is the route requests will use if you follow [my approach to Web API controllers in Sitecore 8](http://www.agehrke.com/2015/08/web-api-controller-in-sitecore-8/).

### 4: Add globally defined filters

Sitecore adds various Web API filters _globally_ to `HttpConfiguration.Filters` based on configuration element `api/services/configuration/filters`. Globally defined filters run for each and every request through Web API. I have [previously blogged about these filters](http://www.agehrke.com/2016/02/disable-sitecore-filters-web-api-controllers/ "Disable Sitecore filters for Web API controllers"), and encouraged Sitecore to use [controller-specific configuration](http://www.asp.net/web-api/overview/advanced/configuring-aspnet-web-api#percontrollerconfig).

### 5: Custom MediaTypeFormatter - BrowserJsonFormatter

Sitecore adds an `BrowserJsonFormatter` _globally_ via `HttpConfiguration.Formatters` which extends the default `JsonMediaTypeFormatter`. It basically ensures that you will get JSON back when making requests to Web API endpoints manually through the browser. However the implementation has one flaw - it hardcodes `Content-Type` response header to `application/json` and removes any charset information set by `JsonMediaTypeFormatter`. This may cause your browser to incorrectly interpret any Unicode characters in the response.

## Suggestions for Sitecore

I believe it's bad practice for a library/platform to replace or modify default behavior of other libraries/platforms - at least in undocumented ways. Doing so can make it difficult for developers to track down misbehavior or errors in a library that would otherwise run without problems. But this is exactly what Sitecore.Services.Client does by changing default `IHttpControllerSelector` and adding filters and MediaTypeFormatters to the global `HttpConfiguration`.

Bullet 5 can be changed to applying needed changes to a local `HttpControllerSettings` using [controller-specific configuration](http://www.asp.net/web-api/overview/advanced/configuring-aspnet-web-api#percontrollerconfig). Changes and implementation should be fairly straight forward. For filters Sitecore could also make sure of controller-specific configuration, and add an implementation of `IFilterProvider` to `HttpControllerSettings.Services`. This implementation would then read filters defined in Sitecore's configuration.

Changing of `IHttpControllerSelector` is however a bit more difficult. We can't use `HttpControllerSettings` as we do not know which controller to select yet. I believe the best place to extend Web API is by creating a custom [`HttpMessageHandler`](http://www.asp.net/web-api/overview/advanced/http-message-handlers). By default `HttpControllerDispatcher`, an implementation of HttpMessageHandler, is used to dispatch a request to a controller based on `HttpControllerDescriptor` returned by `IHttpControllerSelector` - see [source code of HttpControllerDispatcher](https://github.com/ASP-NET-MVC/aspnetwebstack/blob/master/src/System.Web.Http/Dispatcher/HttpControllerDispatcher.cs#L118). I believe Sitecore could copy or extend this class and configure it to use their `NamespaceHttpControllerSelector`. This new `HttpMessageHandler` can be passed to [`MapHttpRoute()`](https://msdn.microsoft.com/en-us/library/system.web.http.httproutecollectionextensions.maphttproute(v=vs.118).aspx#M:System.Web.Http.HttpRouteCollectionExtensions.MapHttpRoute%28System.Web.Http.HttpRouteCollection,System.String,System.String,System.Object,System.Object,System.Net.Http.HttpMessageHandler%29) when defining routes in `DefaultRouteMapper`.

This leaves enabling of CORS which should not interfere with controllers that do not have the `EnableCorsAttribute`.

All in all I hope Sitecore will change their current practice of modifying default behavior of ASP.NET Web API.
