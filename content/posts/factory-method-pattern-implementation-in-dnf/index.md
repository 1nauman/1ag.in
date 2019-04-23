---
title: "Factory method pattern implementation in .net framework"
date: 2019-01-03T20:29:54+05:30
featuredImage: "posts/factory-method-pattern-implementation-in-dnf/images/factory.gif"
---

Factory method pattern also called as Virtual Constructor (less known), defines an interface for creating objects, but defer the instantiation to sub-classes.
<img src="images/factory.gif"/>
Read more about the pattern: [Wikipedia](https://en.wikipedia.org/wiki/Factory_method_pattern), [dofactory](https://www.dofactory.com/net/factory-method-design-pattern), [Sourcemaking](https://sourcemaking.com/design_patterns/factory_method), [oodesign](https://www.oodesign.com/factory-method-pattern.html), etc.

I can safely state, that: A well designed library cannot exist without implementing factory method pattern.

We can see .net framework implementing this pattern extensively, few examples:

- `System.Activator.CreateInstance(Type type)`
- `System.Net.WebRequest.Create(string requestUriString)` based on the url scheme, if url starts with http://, a `System.Net.HttpWebRequest` instance will be returned. For ftp:// a `System.Net.FtpWebRequest`.
- `System.Convert.ToInt32`, `System.Convert.ToBoolean`, etc.
- `System.Web.Mvc.DefaultControllerFactory.CreateController(RequestContext requestContext, string controllerName)` based on `controllerName` parameter, this method returns instance specified.
