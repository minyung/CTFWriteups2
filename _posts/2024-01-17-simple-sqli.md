---
title: simple_sqli
author: sally
date: 2024-01-17 23:24:00 +0800
categories: [Dreamhack, Web]
tags: [web, sql injection, level1]
img_path: /assets/img/posts/20240117/
---

## 문제
![login](simple_sqli/login.jpg){: .w-75 .shadow .rounded-10 w='1212' h='668' }

## 풀이 방법
**app.py**
```python
#!/usr/bin/python3
from flask import Flask, request, render_template, g
import sqlite3
import os
import binascii

app = Flask(__name__)
app.secret_key = os.urandom(32)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

DATABASE = "database.db"
if os.path.exists(DATABASE) == False:
    db = sqlite3.connect(DATABASE)
    db.execute('create table users(userid char(100), userpassword char(100));')
    db.execute(f'insert into users(userid, userpassword) values ("guest", "guest"), ("admin", "{binascii.hexlify(os.urandom(16)).decode("utf8")}");')
    db.commit()
    db.close()

def get_db():
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = sqlite3.connect(DATABASE)
    db.row_factory = sqlite3.Row
    return db

def query_db(query, one=True):
    cur = get_db().execute(query)
    rv = cur.fetchall()
    cur.close()
    return (rv[0] if rv else None) if one else rv

@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    else:
        userid = request.form.get('userid')
        userpassword = request.form.get('userpassword')
        res = query_db(f'select * from users where userid="{userid}" and userpassword="{userpassword}"')
        if res:
            userid = res[0]
            if userid == 'admin':
                return f'hello {userid} flag is {FLAG}'
            return f'<script>alert("hello {userid}");history.go(-1);</script>'
        return '<script>alert("wrong");history.go(-1);</script>'

app.run(host='0.0.0.0', port=8000)
```

문제 파일 코드를 간단히 설명하면 다음과 같다. 

1. **sqlite3 db** 사용
2. users table 생성 (userid, userpassword)
3. ("guest", "guest"), ("admin", random value) 를 삽입
4. **admin** 계정으로 로그인하면 **FLAG** 값을 출력

admin으로 로그인하면 플래그를 획득할 수 있다는 것을 알 수 있다. 

코드의 로그인 확인 쿼리는 다음과 같다.
```sql
select * from users where userid="{userid}" and userpassword="{userpassword}"
```

[sqlite의 주석](https://www.sqlite.org/lang_comment.html)은 "--"이기 때문에 userid를 `admin"--`으로 입력하면 쿼리문이 다음과 같이 될 것 이다. 
```sql
select * from users where userid="admin"--" and userpassword="123"
```

패스워드는 아무거나 입력해도, 성공적으로 플래그를 획득 할 수 있다. 
