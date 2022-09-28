---
title: "MVC REST services in Umbraco 4"
date: "2012-10-24"
categories: 
  - "umbraco"
tags: 
  - "umbraco"
---

Create a Global.asax file for your Umbraco project. In the Application\_Start method register the MVC routes: \[csharp\] protected void Application\_Start() { RouteTable.Routes.MapRoute("ServiceRoute", "Services/{controller}/{action}/{id}", new { controller = "Home", action = "Index", id = UrlParameter.Optional }, new string\[\] { "Your.Namespace.For.Mvc.Controllers" }); } \[/csharp\]

Because Umbraco already contains an empty compiled Global.asax/HttpApplication class in the bin folder, you must delete the "bin/App\_global.asax.dll" file. You might [read other ways](http://blog.mattbrailsford.com/2010/07/11/registering-an-application-start-event-handler-in-umbraco/) to register Application\_Start events in Umbraco. However I never managed to get this to work.

You also need to instruct Umbraco to ignore URLs for the MVC controllers. In Umbraco's web.config's change the "umbracoReservedPaths" app setting: \[xml\] <appSettings> <add key="umbracoReservedPaths" value="~/umbraco,~/install/,~/Services/" /> </appSettings> \[/xml\]
