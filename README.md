# Firefox IOS QR Code Reader XSS (CVE-2019-17003)

One of the most common ways to navigate to a website or URLs is by typing website address in the browser address bar

But this might be frustrating if you have to type a complex web address that includes some kind of tokens.  

And hence QR code scanner comes handy. You just need to scan a QR code and you will be redirected to specified URL or the fetched information will be displayed on your screen. 

Almost every one of us uses random QR codes at so many places, but have you ever wondered what could go wrong if the trusted QR scanner software not implemented correctly?

According to this tweet by firefox

<blockquote class="twitter-tweet"><p lang="en" dir="ltr"><a href="https://t.co/Syz7ijH4a4">https://t.co/Syz7ijH4a4</a><br><br>Biggest update yet! Version 10 has an updated minimalist UI, the ability to hide images, a QR reader, highlights from your previous browsing sessions (unless you’re using Private Browsing!), and trending stories from Pocket.</p>&mdash; Firefox for iOS (@FirefoxiOS) <a href="https://twitter.com/FirefoxiOS/status/930445686671151104?ref_src=twsrc%5Etfw">November 14, 2017</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

QR scanner is supported in firefox since version 10 

> Firefox IOS use WebKit web view to render the content and rest of the part is written in SWIFT by firefox that means the rest of the part comes under the firefox bug bounty program.

Here is a quick image that shows how we can use QR for the navigation

![1](https://github.com/c0d3G33k/Firefox-QR-Code-Reader-XSS/blob/master/ezgif-6-d0a0b9b8ae0b.gif)

So what could go wrong? As an evil mind, the first strategy is to always look for the different `javascript URI` navigation patterns, and hence the first idea is to check what would happen if instead of the URL we make a QR code with `javascript` URI?

Will the browser treat them like normal text? or it will execute as javascript code? Let's figure it out. 

Now, all we need is a website that can generate QR code based on our input I found this website https://www.the-qrcode-generator.com/ pretty easy.

Next, let's navigate to `free text` section of the site and generate a QR code with javascript URI. 

```
javascript:fetch('/').then(res=>res.text()).then(data=>alert(data));
```

It could be more simple in the following way 
```
javascript:alert(document.body.innerHTML);
```
But i like fetch API.

The above code will try to fetch the source of last visited domain by the user.

Let's suppose our user is on google.com After he decided to navigate to another address by scanning the QR code. so in this case as soon as the user scans the QR code source of the current domain will be prompted to the user since firefox evaluated the `javascript URI`.

![2](https://github.com/c0d3G33k/Firefox-QR-Code-Reader-XSS/blob/master/ezgif-6-dbc1446a7aad.gif)

> Now people who scan random QR code on random websites can think of the different consequences.

Since we have javascript code execution on the current domain, we can steal user's data, cookies and what not! 

So If you are thinking this is limited to the web address you might be wrong! Let's explore other places

The first thing that came to my mind is the browser's reader mode

### XSS In Reader Mode
---

> A reader mode is a feature implemented in most of the browsers that allow users to read articles in a clutter-free view i.e rendering a page in a way that will be easy to read without any distraction. During the rendering process, browsers remove all unnecessary code, like javaScript, iframes, other embedding elements etc to make sure there will be no advertisement or any interruption while being in the reading mode.

I tried the same exploit while the document rendered in the reader mode and it worked as demonstrated in the below image

![3](https://github.com/c0d3G33k/Firefox-QR-Code-Reader-XSS/blob/master/ezgif-6-14b3b21118a6.gif)

Did you noticed the javascript code executed in reference to `http://localhost:6571` and if you check `window.location.href` it will be 
```
http://localhost:6571/reader-mode/page?url=https://wwww.anysite.com
```
> Now basically you can you add any URL in the last which makes another vulnerability that allows firefox to put any website in the reader mode even though the reading mode is not available for the particular website. 
---
> You can also look for other possible attack vectors here

### XSS In Local Files

---
Next, we can try the same attack in local files as well, that too worked fine for us as demonstrated in the below image. 
One more thing that we can do here is we can also load the local files by making QR code with local file URI like
```
file:///test.html
```
![4](https://github.com/c0d3G33k/Firefox-QR-Code-Reader-XSS/blob/master/ezgif-6-71759ff3ce82.gif)

### XSS In Internal Pages 
----

Apart from domains, reader mode and local files we can also XSS internal pages of browser in case of firefox they run under 
`internal://` pseudo-protocol.

![5](https://github.com/c0d3G33k/Firefox-QR-Code-Reader-XSS/blob/master/Internal%20protocols.PNG)

### CSP Bypass
----
It turns out this vulnerability also bypass the CSP Let's suppose a website only allowing content that comes from the site's own content as given below
```
// test.php

<?php
header("Content-Security-Policy: default-src 'self'");
if(isset($_GET['xss'])) {

    echo $_GET['xss'];
}
?>
```
even if you are on `test.php` our javascript code will be executed without any restrication.

### Other sources

----
Next, I have tried to find other sources as well, and I found that we can search selected text using the same or other browsers. 
Here also if the selected text is `javascript URI` Firefox will execute javascript code instead of searching it on the default search engine. This particular vulnerability was not useful because the javascript code executed in reference to a blank domain which is harmless as demonstrated in the below image.

![6](https://github.com/c0d3G33k/Firefox-QR-Code-Reader-XSS/blob/master/ezgif-6-42ac9ea4c718.gif)

But still, we can navigate the user to other malicious websites.

### Other browsers

Apart from Mozilla, Opera mini for IOS devices was also affected, We have reported this vulnerability to opera but they have not responded back to us.  

### Root cause 

Till the version 19 firefox was supporting `javascript URI` from the address bar, we can check from the source code 
> [firefox-ios/BrowserViewController+WebViewDelegates.swift at v19.x · mozilla-mobile/firefox-ios · GitHub](https://github.com/mozilla-mobile/firefox-ios/blob/v19.x/Client/Frontend/Browser/BrowserViewController/BrowserViewController%2BWebViewDelegates.swift)

![7](https://github.com/c0d3G33k/Firefox-QR-Code-Reader-XSS/blob/master/Screenshot%20at%20Dec%2002%2015-12-39.png)

After reporting this bug firefox has removed `javascript URI` support from the address bar, as we can see here
>[firefox-ios/BrowserViewController+WebViewDelegates.swift at master · mozilla-mobile/firefox-ios · GitHub](https://github.com/mozilla-mobile/firefox-ios/blob/master/Client/Frontend/Browser/BrowserViewController/BrowserViewController%2BWebViewDelegates.swift)

![8](https://github.com/c0d3G33k/Firefox-QR-Code-Reader-XSS/blob/master/Screenshot%20at%20Dec%2002%2015-10-18.png)


### Advisiory:

https://payatu.com/advisory/firefox-ios-qr-code-reader-xss
