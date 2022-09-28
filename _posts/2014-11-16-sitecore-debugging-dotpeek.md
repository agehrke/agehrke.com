---
title: "Sitecore debugging with dotPeek"
date: "2014-11-16"
categories: 
  - "dotnet-dev"
tags: 
  - "sitecore"
---

Jetbrains dotPeek tool can act as a symbol server for Visual Studio. Read about [how to set it up on their excellent webhelp](https://www.jetbrains.com/decompiler/webhelp/Using_product_as_a_Symbol_Server.html). Now leave dotPeek running and attach the Visual Studio debugger to your w3wp.exe process, and let dotPeek generate PDB files and source files for your Sitecore solution. It can take dotPeek several minutes to decompile all assemblies. Follow the decompilation process in the [Project/PDB Generation Log](https://www.jetbrains.com/decompiler/webhelp/Reference__Project_PDB_Generation.html).

You can now debug into Sitecore assemblies which can come in handy when using one of the many less documented features of Sitecore.
