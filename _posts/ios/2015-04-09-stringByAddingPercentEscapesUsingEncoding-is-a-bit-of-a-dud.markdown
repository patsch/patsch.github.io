---
layout: post
title: "stringByAddingPercentEscapesUsingEncoding is a bit of a dud"
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

I'm writing this little iPhone App for my local soccer club; the idea is that it first identifies the player by scraping some of their details, in particular the club they play for, off the official FFA website,
 then to get the draw & results for that club from another site.

### URL Encoding
I usually go with [AFNetworking](https://github.com/AFNetworking/AFNetworking), but had already gotten a working setup that just used
[<code class="bx">NSMutableURLRequest</code>](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSMutableURLRequest_Class/index.html)
 and [<code class="bx">NSURLSessionDataTask</code>](https://developer.apple.com/library/mac/documentation/Foundation/Reference/NSURLSessionDataTask_class/), so I stuck with them.


 However when I used my usual toolbox to log the user into their MyFootballClub account, I had some issues - the site just wouldn't accept the credentials. I checked HTTP headers, cookies, everything - and ultimately
 found the culprit in an NSString helper method that was supposed to [URL encode](http://en.wikipedia.org/wiki/Percent-encoding) my parameters, but apparently didn't.

### What happens when you submit a form
When you submit a form on a website, the parameters that get sent need to be encoded, so they can be received on the other end the same way you sent them.
The most obvious need for URL encoding concerns the <code class="bx">&</code> character. Say you want to login to a website,
 so when you click the submit button on that site there's likely at least two parameters being submitted: the login, and the password.
The actual HTTP request that is sent includes these parameters all huddled up in the body of the request, i.e. like this:
<pre>
login=foo&password=bar
</pre>
As you can see, the <code class="bx">=</code> characters is used to separate a form parameter name from its value, and the <code class="bx">&</code> is used to separate two form parameters from each other.
This works the same way when you are not sending form parameters via a HTTP POST request, when the parameters are sent in the body of the request, but use a HTTP GET instead.
In that case the form parameters are encoded in the URL itself, as so called query string parameters. In many cases you can send your parameters either way, with the query string being usually easier to accomplish but a lot less safe,
as web servers store the complete URL (including the query string) in their log files:

<pre>
http://www.somewebsite.com/?login=foo&password=bar
</pre>


So if either your parameter name or value includes, for example, a <code class="bx">&</code> or a <code class="bx">=</code> character, then your request is stuffed, as the server you send the request to
can no longer rely on the <code class="bx">=</code> and  <code class="bx">&</code> characters
to perform their separating role. So say your login was not <code class="bx">foo</code> but <code class="bx">=&foo&=</code>, then, without encoding the parameters, the HTTP request would send

<pre>
login==&foo&=&password=bar
</pre>

And how does the server now know which of the &'s belongs to a form parameter, and which separates two form parameters? They can't. Hence the need to encode these characters.

### NSString's stringByAddingPercentEscapesUsingEncoding helper

I knew about the need to URL encode my parameters, so I found out quickly that NSString had a helper that was exactly what I needed:
<blockquote>
stringByAddingPercentEscapes:UsingEncoding - Returns a representation of the receiver using a given encoding to determine the percent escapes necessary to convert the receiver into a legal URL string.
</blockquote>

However after a lot of tracing I noticed that this method did absolutely nothing to my form parameters. The site I am scraping uses ASP.NET, which means all of their forms have
a set of standardised hidden parameters, <code class="bx">__EVENTVALIDATION</code>
being one of them. The value of the __EVENTVALIDATION parameter, which is generated on the server and has to be sent
back with subsequent requests to keep the ASP.NET app happy, usually starts with a forward slash:

<pre>
__EVENTVALIDATION=/QK1qb
</pre>

So we have underscores in the parameter name (__EVENTVALIDATION), and a / in the value -
both of these characters should be encoded, but the stringByAddingPercentEscapes:UsingEncoding method just didn't do it, the string remained completely unchanged.

### stringByAddingPercentEncodingWithAllowedCharacters to the rescue

I then turned my attention to another NSString helper method, that promised similar glory, but with a slightly different parameter. This one allows you to pass in a set of characters
 that you want to leave untouched - and only characters not in that set are encoded.

The <code class="bx">NSCharacterSet</code> class has a convenient factory method called
<code class="bx">URLQueryAllowedCharacterSet</code> - which, according to Apple, returns all characters that are allowed in an URL's query string. However when I tried to modify
the string that way, it remained again unchanged.

Only after I called <code class="bx">stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet alphanumericCharacterSet]</code> on my string were they finally
changed the way they should be - and the login worked straight away.

Here is some test code I wrote to illustrate the issue:

<div class="sunlight-highlight-objective-c">

NSString *name = @"__EVENTVALIDATION";
NSString *value = @"/QK1qb";

// what you'd expect would work
NSString *encodedName = [name stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
NSString *encodedValue = [value stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];


// what you'd also expect would work

NSString *encodedNameURL = [name stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet URLQueryAllowedCharacterSet]];

NSString *encodedValueURL =[value stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet URLQueryAllowedCharacterSet]];

// what does work
NSString *encodedNameGood = [name stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet alphanumericCharacterSet]];

NSString *encodedValueGood = [value stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet alphanumericCharacterSet]];


NSArray *names = @[name,encodedName,encodedNameURL,encodedNameGood];
NSArray *values = @[value,encodedValue,encodedValueURL,encodedValueGood];
NSArray *titles = @[@"Original",@"escaping using UTF Encoding",@"Escaping all but URL Query allowed characters",@"Escaping all but alphanumeric characters"];

for (int i = 0 ; i < [titles count] ; i++)
{
    NSLog(@"\n------------------------\n%@",titles[i]);
    NSLog(@"Name  : %@",names[i]);
    NSLog(@"Value : %@",values[i]);
}
</div>

and the output is this:
<div class="sunlight-highlight-bash">
2015-04-09 11:27:23.961 EscapeTest[77093:4331738]
------------------------
Original
2015-04-09 11:27:23.962 EscapeTest[77093:4331738] Name  : __EVENTVALIDATION
2015-04-09 11:27:23.962 EscapeTest[77093:4331738] Value : /QK1qb
2015-04-09 11:27:23.962 EscapeTest[77093:4331738]
------------------------
escaping using UTF Encoding
2015-04-09 11:27:23.962 EscapeTest[77093:4331738] Name  : __EVENTVALIDATION
2015-04-09 11:27:23.962 EscapeTest[77093:4331738] Value : /QK1qb
2015-04-09 11:27:23.962 EscapeTest[77093:4331738]
------------------------
Escaping all but URL Query allowed characters
2015-04-09 11:27:23.963 EscapeTest[77093:4331738] Name  : __EVENTVALIDATION
2015-04-09 11:27:23.963 EscapeTest[77093:4331738] Value : /QK1qb
2015-04-09 11:27:23.963 EscapeTest[77093:4331738]
------------------------
Escaping all but alphanumeric characters
2015-04-09 11:27:23.963 EscapeTest[77093:4331738] Name  : %5F%5FEVENTVALIDATION
2015-04-09 11:27:23.963 EscapeTest[77093:4331738] Value : %2FQK1qb
</div>
