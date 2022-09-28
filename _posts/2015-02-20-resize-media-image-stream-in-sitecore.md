---
title: "Resize media image stream in Sitecore"
date: "2015-02-20"
categories: 
  - "dotnet-dev"
tags: 
  - "sitecore"
---

When working with MediaItems in Sitecore you might want to direct access to a stream behind the media item. On the a `Sitecore.Data.Items.MediaItem` you can call `GetMediaStream()` to accomplish that. It will return a stream to the raw data which might not be ideal if you are working with images. Often you would want a resized image, and fortunately you can use the `Sitecore.Resources.Media.ResizeProcessor` pipeline processor.

<script src="https://gist.github.com/agehrke/c9fbef951bd520db7ec6.js"></script>
