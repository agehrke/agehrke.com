---
title: "Web API error in Sitecore 8.1 Update-2 on CD-servers"
date: "2016-04-27"
categories: 
  - "dotnet-dev"
tags: 
  - "sitecore"
---

This post describes a problem with Web API in Sitecore 8.1 Update-2, causing all Web API controllers to return _"An error has occurred."_ error message on CD-servers. You can skip to [my temporary solution](#solution) at the end, if you experience the same issue.

I have previously written about Sitecore.Services.Client and use of ServicesControllerAttribute for easy implementation of Web API in a Sitecore solution - without any pipeline and routing configuration. After upgrading to Sitecore 8.1 Update-2, and reapplying the tedious manual work of [enabling/disabling config files for CD-servers](https://doc.sitecore.net/sitecore_experience_platform/setting_up__maintaining/xdb/configuring_servers/configure_a_content_delivery_server), I found that all our Web API controllers failed with a _An error has occurred_ error message. Here's the full exception message returned by Web API after disabling `customErrors` in web.config:

<script src="https://gist.github.com/agehrke/54593bf1c2abb42d8d002abf7bfe1047.js"></script>

What first caught my eye was that the stack trace contained non of our custom code, and that the error was happening deep inside Sitecore and Web Api. I was hitting a custom controller and not anything related to _ExperienceAnalytics_, which the stack trace could indicate at first glance.

Here's what I wrote in a ticket to Sitecore Support:

From the exception stack trace I have narrowed down the issue to `Sitecore.Services.Infrastructure.Web.Http.Dispatcher.NamespaceHttpControllerSelector.InitializeControllerDictionary()`, which instantiates a `HttpControllerDescriptor` for each API controller found in _any_ referenced assembly. Some API controllers in the `Sitecore.ExperienceAnalytics.dll` assembly are decorated with `Sitecore.ExperienceAnalytics.Api.Http.Filters.NotFoundExceptionFilterAttribute`. This attribute contains a parameterless constructor that is called by internals of ASP.NET Web API. However the constructor calls `Sitecore.ExperienceAnalytics.Api.ApiContainer.GetLogger()` which tries to create an object based on what is defined in an `experienceAnalytics/api/logger` configuration element. This element does not exist as it's defined in `Sitecore.ExperienceAnalytics.WebAPI.config`, which should be disabled on CD servers.

### My (temporary) solution

I have resolved the issue by deleting three Sitecore.ExperienceAnalytics assemblies on CD-servers:

- Sitecore.ExperienceAnalytics.Client.dll
- Sitecore.ExperienceAnalytics.dll
- Sitecore.ExperienceAnalytics.ReAggregation.dll

This prevents the code in `InitializeControllerDictionary()` to find and instantiate the problematic controllers and filters for ExperienceAnalytics.

### Update May 11: Sitecore documentation updated

On May 11th I received information Sitecore Support that they had updated [documentation about configuring CD servers](https://doc.sitecore.net/sitecore_experience_platform/setting_up__maintaining/xdb/configuring_servers/configure_a_content_delivery_server). Now, this article has the following:

> ### xAnalytics assembly files
> 
> If you have implemented custom code that uses ASP.NET Web API attribute routing, to avoid errors we recommend that you disable the following .dll files in the `\Website\bin` folder:
> 
> - Sitecore.ExperienceAnalytics.dll
> - Sitecore.ExperienceAnalytics.Client.dll
> - Sitecore.ExperienceAnalytics.ReAggregation.dll
