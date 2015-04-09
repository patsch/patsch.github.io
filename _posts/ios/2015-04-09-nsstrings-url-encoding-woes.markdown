---
layout: post
title: "NSString's URL Encoding Woes"
modified:
categories: ios
excerpt: When trying to screenscrape a website from within an iPhone app, I found that everything is not as it seems with NSString's URL encoding capabilities.
tags: [ios,objective-c,nsstring,http,url encoding,hpple]
image:
  feature: grab-www.jpg
  credit: Prestashop
  creditlink: https://fmemodules.wordpress.com/category/prestashop-seo/
date: 2015-04-09T11:29:06+10:00
---

<pre>
 <code class="objective-c">
-(void)loginWith:(NSString*)login
{
  return [NSString stringWithFormat:@"bla"];
}
 </code>
</pre>
