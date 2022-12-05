---
layout: post
title: Intigriti's Nov XSS Challenge Writeup
subtitle: XSS 
cover-img: /assets/img/wsc.jpg
thumbnail-img: /assets/img/wsc.jpg
share-img: /assets/img/wsc.jpg
tags: [xss]
---

# Intigriti's Nov XSS Challenge - writeup

> Challenge URL: https://challenge-1122.intigriti.io/

Basically, this is also a note-taking application like my [previous month's challenge](https://challenge-1122.intigriti.io/). The goal of the challenge is to take over the admin's account which has the flag in it. 

---

## Intro

* This Challenge is Organized in a bit ~~different~~ weird way. 
* The Notes Application is running on `api.challenge-1122.intigriti.io` and `cdn.challenge-1122.intigriti.io` is used to store the notes and profile pictures of users
* Interestingly `cdn.challenge-1122.intigriti.io` uses Varnish to cache static files
* Also, the Javascript code in `api.challenge-1122.intigriti.io` reflects one subdomain `staging.challange-1122.intigriti.io`, and staging the domain also runs the same application. But Here is an ATO vuln. Basically, JWT used to sign the token are same for staging and the main domain. So, we can register any username on `staging` and use the signed JWT on `api.challenge-1122.intigriti.io` to take over any account. (admin username is unpredictable)

## Notes

* Notes are stored in the pattern of `cdn.challenge-1122.intigiti.io/<username>-<uuid>.html`
* Also, CSP Present in the Response Header `Content-Security-Policy: script-src 'none'; object-src 'none'`. So, Javascript executions will be blocked 

![](https://i.imgur.com/8dbkeUv.png)

## Profile Pic Caching

* As you can see, the below picture reflects that profile pictures are cached and served from cache `X-Cache: HIT`

![](https://i.imgur.com/KYwaACv.png)


* After playing a bit, found that vanish caching the static files, and it identifies "Static files" if the extension is `.png` or `.jpg`  ...


## Caching the Uncached 

* By sending the below request 2 times, we can clearly see, this request is identified as a "static file" and cached by varnish.

![](https://i.imgur.com/sUIOFCH.png)

* Surprisingly, there is no CSP Here. So, By Abusing this, it is possible to Exploit XSS here. 


![](https://i.imgur.com/pOEz5rJ.png)

![](https://i.imgur.com/chF6Pq7.png)


## Exploit! 

* I found 2 possible solutions. 
    * Use the Above XSS to Register a Service Worker on `cdn.challenge-1122.intigriti.io` to cache all requested pages. =>  redirect the Admin bot to `api.challenge-1122.intigriti.io` and when this page loads, notes created by the admin bot will be loaded into the page => service worker caches the pages => send the cached URLs over to attacker's site

    * Another Solution is, with the XSS in `cdn.challenge-1122.intigriti.io`, calling `window.open("https://api.challenge-1122.intigriti.io")` and when the `api.challenge-1122.intigriti.io` loads, it will load all posts create by the user (admin bot here) by framing the `cdn.challenge-1122.intigriti.io/<username>-<uuid>.html` => Here, `api.challenge` is a child and `cdn.challenge` is the parent. So, it is possible to read the frames src from `cdn.challenge` on `api.challenge` if the frame-src is the same origin as `cdn.challenge`. 


![](https://i.imgur.com/ieRIQFV.png)

* After leaking the iframe link, we can find the username of the admin, we can create an account on `staging` domain with that username => get signed JWT => use that JWT on `api.challange-1122.intigriti.io` to the takeover admin account. The flag is on Admin's profile pic. 


> INTIGRITI{workinghardorhardlyworking?}
