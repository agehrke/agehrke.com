---
title: "Setting Content-Type in an Umbraco /Base service"
date: "2011-09-12"
categories: 
  - "umbraco"
tags: 
  - "umbraco"
---

The [Umbraco /Base system](http://our.umbraco.org/wiki/reference/umbraco-base) is great for creating simple REST-like services. You can use the RestExtensionMethod attribute to instruct Umbraco to return your response as XML or not. But what is “not”? Apparently it means set Response.ContentType to “text/html”. But what if the returned data should be sent using a different mime type? Setting the ContentType manually like

\[csharp\] HttpContext.Current.Response.ContentType = "application/json" \[/csharp\]

will not do the trick. Umbraco /Base will override it with “text/html” if you are returning a string from your service method.

What you need to do is to change your method return signature to “void” and write your data directly to the HttpResponse, like: \[csharp\] HttpContext.Current.Response.ContentType = "application/json"; HttpContext.Current.Response.Write(stringWriter.ToString()); \[/csharp\]

Umbraco will not override the content type for “void”-methods. Below is a full example.

\[csharp\] \[RestExtension("Member")\] public class MemberService { \[RestExtensionMethod(returnXml = false, allowAll = true)\] public static void Json() { var json = "{ \\"firstName\\": \\"John\\", \\"lastName\\": \\"Smith\\", \\"age\\": 25 }";

// Write response and set content type HttpContext.Current.Response.ContentType = "application/json"; HttpContext.Current.Response.Write(json); } } \[/csharp\]
