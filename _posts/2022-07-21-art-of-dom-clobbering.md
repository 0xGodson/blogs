
---
layout: post
title: An Art of Dom Clobbering - From Zero to Advance Level
subtitle: Clobbering 
cover-img: /assets/img/wsc.jpg
thumbnail-img: /assets/img/wsc.jpg
share-img: /assets/img/wsc.jpg
tags: [Research]
---



# Agenda

* What is Dom Clobbering?
* Basics of Dom Clobbering
* Writing Window Object
* Writing Document Object 
* Why it's not possible to Override Window Object and Why it is possible to override document Object
* Javascript Prototype Chain
* Clobbering More that 1 Level
* Conclusion

## Prerequisites - HTML, Good Understanding of Javascript and Javascript Prototype


# What is Dom Clobbering? 

* DOM clobbering is a technique in which you inject HTML into a page to manipulate the DOM and ultimately change the behavior of JavaScript on the page. DOM clobbering is particularly useful in cases where  [XSS](https://portswigger.net/web-security/cross-site-scripting)  is not possible, but you can control some HTML on a page where the attributes  `id`  or  `name`  are whitelisted by the HTML filter. The most common form of DOM clobbering uses an anchor element to overwrite a global variable, which is then used by the application in an unsafe way, such as generating a dynamic script URL. - PortSwigger

# Basics of Dom Clobbering
### Before getting into Dom Clobbering, we need to learn about `id` attribute and `name` attribute

## ID Attribute
* The id attribute **specifies its element's unique identifier (ID)** . The value must be unique amongst all the IDs in the element's home subtree and must contain at least one character. The value must not contain any space characters

* Ex: 

 <img src="https://i.imgur.com/WiU6Zeb.png">


## Name Attribute
* Basically  `name`  attribute specifies a name for an HTML element.
* The name attribute  **specifies a name for an HTML element**. This name attribute can be used to reference the element in a JavaScript. For a <form> element, the name attribute is used as a reference when the data is submitted. For an `<iframe>` element, the name attribute can be used to target a form submission.
* Ex:

*<img src="https://i.imgur.com/ec1lNSg.png">



##  Basic Stuffs

### Introduction to Javascript Objects
An object type is simply  **a collection of properties in the form of name and value pairs**. Notice from the list that null and undefined are primitive JavaScript data types, each being a data type containing just one value.
* Ex: 

<img src="https://i.imgur.com/SEIHBSg.png">



### What is Window in Javascript?
The window object is supported by all browsers. **It represents the browser's window**. All global JavaScript objects, functions, and variables automatically become members of the window object. Global variables are properties of the window object. Global functions are methods of the window object.

### What is document in javascript?
The document object **represents your web page**. If you want to access any element in an HTML page, you always start with accessing the document object.

---

 * Basically we can Declare Variables in javascript in 3 methods. `var`, `let`, `const`. Variables Declared with `var`  are stored under window object with key as variable name and value as declared variable's value

 <img src="https://i.imgur.com/dGWfLnx.png">

 * We can see, `myName` is a Key in window Object Now.

# Writing Window Objects
* Now, We have Basic understanding of `id`,`name`, `objects`. Now Let's have a Look in Clobbering window Elements.
* As we Seen Already, while used `id` attribute in HTML tags, we can get those tags by `document.getElementById("<id>")` or `window.<id>`. 
* For Ex, If we have a HTML tag `<h1 id="hi">Hey!</h1>`, then we can get this tag by calling `document.getElementById("hi")` or simply call `window.hi`.

## Exploitation: 
### Hidden `toString` function call:
* Sometimes in Javascript,when some functions like `toString` are called hiddenly when some functions are called. those functions use `toString` function internally for some purpose.
Some Examples: 


<img src="https://i.imgur.com/xImWbJj.png">


* In the Above Image, I innerHTMLed a Array into the `body` tag. But the Values are Not appended as List, But converted into String. 

<img src="https://i.imgur.com/ZRRnpf3.png">


* Here Again, When Using template String Syntax, `toString` is automatically called!
 
 
* Another Example, `Array.join` called automatically with using `Array.toString`. There are tons of examples around there. 
 
 
---


* Now, How this Hidden `toString` call in `innerHTML` going to help us?
* Well, If you found a Reflected HTMLi, then this doesn't matter. But incase of DOM based HTMLi, then We need the Help of this hidden `toString` call in order to do something.
* Let's Take a Example:
 
<img src="https://i.imgur.com/rFhJfGn.png">
 
* Here, I called `toString` function for an `h1` tag which return -> `[object HTMLHeadingElement]`
* So, We can't actually Declare our own desire value, but still we can create a `key` under window Object. 
* A Small Research can help us to find a Interesting behaviour amoung tags.

```javascript
var allTags = ["a","img",...]
for (tag of allTags){
    temp = `<${tag} id="test">`;
    document.body.innerHTML = temp;
    try{
        if(!document.getElementById("test").toString().includes("[object")){
        console.log(tag)}
    }catch{}
}
```


* If we Run the About Javascript, We can find that, `a` tag and `area` tag doesn't return `[object...` when `toString` is called. Something interestingly happing with those 2 tags.
* when we Supply a `href` attribute with these tags, we can see, the value of the `href` is returned when `toString` is called. 

<img src="https://i.imgur.com/nJDLIRZ.png">


* Pretty Cool. But still we cannot control full value. `protocol` is messed up. But still we can use `data` uri, `javascript` uri, other cool protocols like `cid` and more.

* We know, every `id` attributes can create a `key` in window object. what about `name` attribute?
* Let's Fuzz Again!

```javascript
var allTags = ["a","img",...]
for (tag of allTags){
    temp = `<${tag} name="test">`;
    document.body.innerHTML = temp;
    try{
        if(window.test){
        console.log(tag)}
    }catch{}
}
```
<img src="https://i.imgur.com/O9WANGy.png">


* Turns out, `embed`,`form`,`iframe`,`image`,`img`,and `object` tags can create a `key` under `window` object.
# Writing document Objects

* Basically `document` object is not like `window` object that stores variables, functions...
* The document object **represents your web page**. If you want to access any element in an HTML page, you always start with accessing the document object.
* So, How can we clobber `document` object? Well, Lets Do a Bit of Research.

```javascript
var allTags = ["a","img",...]
for (tag of allTags){
    temp = `<${tag} id="test">`;
    document.body.innerHTML = temp;
    try{
        if(document.test){
        console.log(tag)}
    }catch{}
}
```
<img src="https://i.imgur.com/uhcMjQs.png">


* Yeah!, We can Clobber document Object with `object` tag with `id` as attribute

<img src="https://i.imgur.com/VCYAF9y.png">


* Pretty Cool right? Now lets try with `name` attribute :)

```javascript
var allTags = ["a","img",...]
for (tag of allTags){
    temp = `<${tag} name="test">`;
    document.body.innerHTML = temp;
    try{
        if(document.test){
        console.log(tag)}
    }catch{}
}
```

<img src="https://i.imgur.com/oMGid0L.png">

 * Turns out, `embed`,`form`,`iframe`,`image`,`img`,and `object` tags can create a `key` under `document` object.
* Well, in `window` clobbering, we used `a` tag and `area` tag with `href` attribute to return our own value. Let See if we can control the output. 

```javascript
var mytags = ["embed","form","iframe","image","img","object"]
for (tag of mytags){
    temp = `<${tag} name="test">`;
    document.body.innerHTML = temp;
    console.log(document.getElementsByName("test").toString())
}
```

<img src="https://i.imgur.com/cXRmWJr.png">

 
* Nope, we can't find anything. So, Its not possible to return our own value.
 
 
--- 

# Why it's not possible to Override Window Object and Why it is possible to override document Object?

* Maybe you can have a question in your mind. why not overwrite the variables already declared?
* Well, We cannot override/overwrite window `keys` with `id`  attribute or `name` attribute. But it is possible to override `document` keys. 
* So, to understand this answer for this question, you must need good understanding about javascript `prototype chain`

# Javascript Prototype Chain
* Prototype type chain is little bit huge topic. So i can't cover that here. 
* You can learn that from <a href="https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain">here</a>. If you kow Tamil, you can refer my OWASP session about Prototype Pollution. There I gave a Detailed View about `prototype chain`. you can watch by clicking <a href="https://www.youtube.com/watch?v=54GYCl7Beh4">here</a>
* Now, Lets Take this example:

<img src="https://i.imgur.com/EvUHpPq.png">


* we have a key `name` in the Object `myObj`. But we Don't have a key `toString` in the Object `myObj`. where the `toString` comes from? It's from prototype of the Object. As i said before, I am not going to explain this topic. you can learn from the above links.
* Cool, In Javascript, there is method/function which help us to find, if the key is really present on the object.

<img src="https://i.imgur.com/WvucheL.png">


* We can see, `hasOwnProperty("name")` returned true and `hasOwnProperty("toString")` returned false.
* So, the `name` is a `key` in `myObj`, `toString` is a function comming from the `Object.prototype`
* what if we mentioned `toString` as a `key` in `myObj`?

<img src="https://i.imgur.com/vlAKxlC.png">


* So, Did we Overwritten the `toString` function same from `Object.prototype`? No. 

<img src="https://i.imgur.com/Cu8K2gw.png">


* You can see, Still we can access the Original `toString` function which printed `[object Object]`.

Edit:

* After some time, <a herf="https://twitter.com/IvarsVids">Ivar Vids</a> texted that, `myObj.__proto__.toString` is not the correct way of calling,and  mentioned to use `Object.prototype.toString.call(myObj)` instead of this. kudos to Ivar Vids
* Why we should call the toString like `Object.prototype.toString.call(myObj)`? Well, Before Getting into that We need to know little bit about `toString`. You can completly Skip this if you only interested in Dom Clobbering
* Most Common Data types in javascript are `String`,`Array`, `Number`.

* Basically `Object`, `Number`,`Array` are build-in Constructor Functions which can be used to Create 
`Objects`, `Numbers`, `Arrays` 

<img src="https://i.imgur.com/lWBd0Vi.png">


* The Below Images Shows that, all three Constructor Objects have a property `toString` in its prototype

<img src="https://i.imgur.com/8lViXrI.png">


* This is Tricky one, If you Look Closed to the image, `hasOwnProperty` is coming from `Object.prototype`. We already Saw, How `hasOwnProperty` works. 
* Even the `hasOwnProperty` coming from `Object.prototype` it is searching for a `key` `toString` in `Object.prototype`.
* To Understand Better

<img src="https://i.imgur.com/L1k9u1l.png">

* Here, We Have a Nested Object. `myobj2` is also a Object, so its Prototype is also equal to `Object.protoptype`
* Another Example with Arrays:

<img src="https://i.imgur.com/8glGFps.png">

* Basically `includes` is a property comimg from its prototype [`Array.prototype`].`inclueds` will return `true/false` values. `true` of passed argument is present in the array, else it will return false. 
* Even `myarr.__proto__.includes` is equal to `myarr.includes`, whiles using `myarr.__proto__.includes(2)`, the `includes` in not searching inside `myarr` but searching inside `myarr.__proto__`
* Lets add a value to `Array.prototype`

<img src="https://i.imgur.com/wlelcIg.png">

* Now, You can clearly see, how the includes works. `toString` is also works similar to this. Even if we call the `toString` from its prototype with `__proto__`, to `toString` is not actually working with the `myObj` but with `myObj.__proto__` when we call `myObj.__proto__.toString`. 
* When we Call `String.prototype.toString`, it will return as a `String`, If we call `Arrar.prototype.toString`, then it will convert the array to `string`. If we call the `Object.prototype.toString`, then it will return `[object <DATA TYPE>]`.

<img src="https://i.imgur.com/s5o5oF7.png">

* So, when we call `myObj.__proto__.toString()` we are not actually working with `myObj`
 but working with `myObj.__proto__`.  `myObj.__proto__.toString()` == `Object.prototype.toString.call(myObj.__proto__)
`
--- 
* Lets Get Back to the Question...

## Why it's not possible to Override Window Object and Why it is possible to override document Object?

### Why we can't override window clobbering?
* To Understand, Why it is not possible to override `window` object clobbering, we need to know where and how the elements and variables are stored in the window Object. 

<img src="https://i.imgur.com/tYT7BdL.png">


* So, if the `h1` tag `id` attribute `whoami` is not stored in `window` object, then how is it possible to access `window.whoami`?
* well, that's called `prototype chain`. While we try to get `window.whoami`, In background,  Javascript will follow its prototype chain to find the `whoami` key. 

<img src="https://i.imgur.com/RWlKDfv.png">


* Now, the `tags` with `id` are stored into it's prototype. it is not stored directly in the `window` object. 
* We already saw, variables/functions are stored in the `window` object itself. it is not stored under its `prototype`. So, when there is a variable already declared, then we can't override that. but still we can access through `prototype chain`

<img src="https://i.imgur.com/3bxS5ER.png">


* while javascript can't find the `whoami` from first chain `__proto__`, then it will automatically move to next chain untill the `__proto__` hit `null`.

 * This is the Reason why we can't override `window` clobbering

 
 ---

## Why we can override document clobbering?
* we can use any one of the above `tags` found to be helpful in document clobbering. for now, we are going to use `img` tag with `name` attribute.
* There is a another method in javascript which help us to view the `keys` from a `object`. methods/functions from the prototype will not be returned. `Object.getOwnPropertyNames(<Object>)`

<img src="https://i.imgur.com/VuGtzes.png">


* Surprisingly, only `location`, `cookie` are it own properties (keys).
* So, where the `document.domain`, `document.getElementById`,.... are comming from? yes its from its prototype!
* So, If we can add `domain` as `key` to `document` object, when we call `document.domain`, instead of the real `document.domain`, our value will be returned. 
* So, let now see where and how the values we inject with `img` tag with `name` attributes are stored.

<img src="https://i.imgur.com/L6bIAsp.png">


* Now, we can see out keys are directly stored into document as it own property.
* So, According to `prototype chain`, our tag overrided the orginal property from its prototype.
* Except `document.location`, we can clobber any thing in `document` Object. This mostly help us to bypass IF Conditions like `if(document.domain != "..."){}` and maybe we can break the functionalities by overwriting over used tags!
 
* Example:
 
 
 <img src="https://i.imgur.com/nxYbcJe.png">

---

# Clobbering More that 1 Level

* Till now, We saw dom clobbering level 1. Like clobbering single property. But actually we can clobber multiple levels like a `Object`. 
* What happen if we used same `id` 2 times for 2 different tags?
 
<img src="https://i.imgur.com/fQiMj9l.png">


* So, basically when we used 2 times same `id`, it create a `HTMLCollection`. We can combine `id` and `name` attributes to build more complex `HTMLCollection`.
* For Example we can use `form` tags with `input/output` tags
* Why form tags? basically they are nested HTML tags.  [Dom Tree Struct]

<img src="https://i.imgur.com/hVcggGa.png">


* we can't call our own attribute which we created while injecting HTML. This method of Calling out Own `attributes` are Exploited in the Past By Legends in `IE`. But it is not possible now.
* Later Terjanq came up a trick which can clobber 4 levels with `iframe` tag and `name` and `id` attributes with nested `iframes` with `srcdoc` attribute. 

<img src="https://i.imgur.com/yKNVu9o.png">


* Basically using `name` attribute with `a` as value creates reference in `window` as `window.a` and `srcdoc` attribute can be used to inject `html` contents.
* In the Nested `iframe` we used `name` attribute and set that to `b`. So we can access that through `window.a.b`.Inside the nested frame we using another `srcdoc` and we using 2 `id` attributes with same value as `c` which creates a `HTMLCollection`. 
* Now, we can access `window.a.b.c`, adding a `name` attribute inside `a` tag, we achive 4 level clobbering .`window.a.b.c.d`

---

# Conclusion
* If you want to pratice this, you can check my challenge at https://0xgodson.me/my-ctfs/chal1/ and postswigger Labs too!


## Fix

-   Check that objects and functions are legitimate. If you are filtering the DOM, make sure you check that the object or function is not a DOM node.
-   Avoid bad code patterns. Using global variables in conjunction with the logical OR operator should be avoided.
-   Use a well-tested library, such as DOMPurify, that accounts for DOM-clobbering vulnerabilities.

# Resources

https://portswigger.net/research/dom-clobbering-strikes-back
https://portswigger.net/web-security/dom-based/dom-clobbering

---
