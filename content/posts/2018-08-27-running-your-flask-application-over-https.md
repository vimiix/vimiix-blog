---
title: '[译]通过HTTPS协议运行你的Flask程序'
date: 2018-08-27 17:22:48
categories: 'HTTPS'
tags: ['HTTPS', 'SSL', 'Flask', 'translation']
---

我们在开发 Flask 应用程序时，通常通过运行 Flask 自带的 Web 服务器来开发测试，这个服务器提供了基本的但功能完备的 WSGI 服务器。但开发结束以后，在应用程序上线到生成环境时，有很多不得不考虑的事情，其中之一是我们是否应该要求客户端使用加密连接以增加安全性。

人们总是问我这个问题，特别是如何在 HTTPS 协议上部署 Flask 服务器。在本文中，我将介绍几种为 Flask 应用程序添加加密的方案，从一个非常简单的可以在五秒内实现，到一个强大的就像我的网站一样可以得到一个 A +评级解决方案（[我的网站的 SSL 分析数据](https://www.ssllabs.com/ssltest/analyze.html?d=blog.miguelgrinberg.com)）。

<!--more-->

![http://ww1.sinaimg.cn/large/8d56d744ly1fuoapa21euj20hh06ajri.jpg](http://ww1.sinaimg.cn/large/8d56d744ly1fuoapa21euj20hh06ajri.jpg)

## HTTPS 是如何工作的？

HTTP 的加密和安全功能是通过[传输层安全性（TLS）](https://en.wikipedia.org/wiki/Transport_Layer_Security)协议实现的。实际上，TLS 定义了一种使任何网络通信信道安全的标准方法。因为我不是一个安全专家，如果我试着给你一个 TLS 协议的详细描述，我认为我无法做到很好，所以我只想告诉你一些我们比较感兴趣的内容，要知道我们给 Flask 服务器设置安全和加密目的。

一般的思路是，当客户端与服务器建立连接并请求加密连接时，服务器将使用其 SSL 证书进行响应。证书此时作为服务器的标识，因为它包括服务器名和域名信息。为确保服务器提供的信息正确无误，证书是由证书颁发机构或 CA 进行加密签名的。如果客户端知道并信任 CA，则可以确认证书签名确实来自该实体，并且客户端可以确定其连接的服务器是合法的。

客户端成功验证证书后，会创建用于与服务器通信的加密密钥。为确保将此密钥安全地发送到服务器，它使用服务器证书附带的公钥对其进行加密。服务器拥有与证书中的公钥一起使用的私钥，因此它是唯一能够解密包的一方。从服务器收到加密密钥时起，所有流量都使用只有客户端和服务器知道的密钥加密。

从上述总结中你可能会猜到，要实现 TLS 加密，我们需要两个项目：服务器证书，包括公钥并由 CA 签名，以及与证书中包含的公钥一起使用的私钥。

## 最简单的方法

Flask(更具体地说其实是**Werkzeug**)，支持使用即时证书，这对于通过 HTTPS 快速提供应用程序非常有用，而且不会搞乱系统的证书。你只有需要做的就是将 `ssl_context ='adhoc'` 添加到程序的 `app.run()` 调用中。遗憾的是，Flask CLI 无法使用此选项。举个例子，下面是官方文档中的“Hello，World” Flask 应用程序，并添加了 TLS 加密：

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run(ssl_context='adhoc')
```

要在 Flask 中使用临时证书，你需要在虚拟环境中安装一个相关依赖项：

```bash
$ pip install pyopenssl
```

当你运行脚本时，就会注意到 Flask 指示它正在运行 `https://` 服务器：

```bash
$ python hello.py
 * Running on https://127.0.0.1:5000/ (Press CTRL+C to quit)
```

很简单，对吧！这样有一个问题就是浏览器不喜欢这种类型的证书，因此浏览器会显示一个大而可怕的警告，你需要在访问应用程序之前先手动允许才可以访问。一旦允许浏览器连接，你就拥有了一个加密连接，就像从具有有效证书的服务器获得的连接一样，这使得这些临时证书便于开发中进行快速和验证测试，但不能用于任何实际生产环境中。

## 自签名证书

所谓的自签名证书是使用与同一证书关联的私钥生成签名的证书。我在上面提到客户端需要“知道并信任”签署证书的 CA，因为这个信任关系允许客户端验证服务器证书。Web 浏览器和其他 HTTP 客户端一般都预先配置了一个已知和可信 CA 的列表，但显然如果你使用自签名证书，CA 将不会被知道，验证也必将失败。这正是我们在上一节中使用的临时证书所发生的情况。如果 Web 浏览器无法验证服务器证书，它将允许你手动来继续并访问相关网站，但这将确保你自己必须了解你所承担的风险。

![](http://ww1.sinaimg.cn/large/8d56d744ly1fuobquio2wj217u0ugacw.jpg)

但是，到底风险是什么呢？使用上一节中的 Flask 服务器来说，显然我们是相信自己的，所以对我们没有任何风险。主要问题是当用户在连接到他们不了解或不是他们控制的站点时出现此警告。在这些情况下，用户无法知道服务器是否可信，因为任何人都可以为任何域生成证书，正如接下来你要看到的。

虽然自签名证书有时很有用，但 Flask 的临时证书并不是那么好，因为每次服务器运行时，都会通过 pyOpenSSL 动态生成不同的证书。使用自签名证书时，最好每次启动服务器时都使用相同的证书，因为这样可以将浏览器配置为信任它，从而消除了安全警告。

我们可以从命令行很轻松生成自签名证书。只需要安装了[openssl](https://www.openssl.org/)：

```bash
openssl req -x509 -newkey rsa:4096 -nodes -out cert.pem -keyout key.pem -days 365
```

这个命令在`cert.pem`中写入新证书，其中相应的私钥位于`key.pem`中，有效期为 365 天。运行此命令时，系统会询问你几个问题（译者注：用于生成证书的所有者信息）。下面你可以看到我如何回答它们为我自己的私有服务器 生成证书：

```bash
Generating a 4096 bit RSA private key
......................++
.............++
writing new private key to 'key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Beijing
Locality Name (eg, city) []:Beijing
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Vimiix
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:vimiix
Email Address []:contact@vimiix.com
```

我们现在可以在 Flask 应用程序中使用这个新的自签名证书，方法是将 `app.run()` 中的 `ssl_context` 参数设置为具有证书和私钥文件的文件名的元组:

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run(ssl_context=('cert.pem', 'key.pem'))
```

浏览器将继续给出警告这个证书，但如果你检查它，你就将看到你在创建时输入的信息：

![](http://ww1.sinaimg.cn/large/8d56d744ly1fuoc3eiccuj20qw0tcq5m.jpg)

## 用于生产的 Web 服务器

当然我们都知道 Flask 开发服务器只适合开发和测试。那么我们如何在生产服务器上安装 SSL 证书呢？

如果您使用的是[gunicorn](http://gunicorn.org/)，可以使用命令行参数执行此操作：

```bash
$ gunicorn --certfile cert.pem --keyfile key.pem -b 0.0.0.0:8000 hello:app
```

如果你使用 nginx 作为反向代理，那么你可以使用 nginx 配置证书，然后 nginx 可以 “全权代理” 加密连接，这意味着它将接受来自外部的加密连接，然后使用常规的未加密连接与我们的后端服务交互。nginx 的配置项如下：

```bash
server {
    listen 443 ssl;
    server_name example.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    # ...
}
```

还需要考虑的另一个重要问题是如何让 nginx 处理通过普通的 HTTP 连接的客户端。在我看来，最好的解决方案是通过重定向到相同的 URL，使用 HTTPS 来响应未加密的请求。对于 Flask 应用程序，您可以使用[Flask-SSLify](https://github.com/kennethreitz/flask-sslify)扩展来实现。对于使用 nginx，我们可以在配置中包含另一个服务器模块：

```bash
server {
    listen 80;
    server_name example.com;
    location / { # 全部转发到https的相同url下
        return 301 https://$host$request_uri;
    }
}
```

如果你使用的是其他 Web 服务器，请查看其文档，应该会找到类似的方法来创建上面显示的配置。

## 使用“真实”证书

我们现在已经了解了自签名证书的所有可选方案，但在所有这些情况下，有共同的限制就是 Web 浏览器仍然不会信任这些证书，除非你手动接受他们，因此生产服务器上的证书的最佳选择站点应该是从一个众所周知并且由所有 Web 浏览器自动信任的 CA 中获取一个证书。当你从某个 CA 请求证书时，这个机构将会验证你是否可以控制你的服务器和域，但验证的执行方式取决于 CA。如果你的服务器通过验证，则 CA 将为你的服务器颁发具有该 CA 签名的证书，并将其提供给你进行安装。证书一般在一段时间内有效，通常不会超过一年。大多数 CA 都会为这些证书收取费用，但有一些 CA 会免费提供这些证书。最受欢迎的免费 CA 就是 ：[Let's Encrypt](https://letsencrypt.org/)。

从 Let's Encrypt 获取证书相当容易，因为整个过程是自动化的。假设你使用的是基于 Ubuntu 的服务器，则必须首先在服务器上安装其开源 certbot 工具：

```bash
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install certbot
```

现在你已准备好请求证书的前置工作了。certbot 有几种方法可用于验证你的网站。通常，“webroot” 方法最容易实现。使用此方法，certbot 会在 Web 服务器公开的目录中添加一些文件作为静态文件，然后尝试使用你希望为其生成证书的域名，通过 HTTP 访问这些文件。如果此测试成功，则 certbot 知道运行它的服务器与正确的域名相关联，并且满足并发布证书。使用此方法请求证书的命令如下

```bash
$ sudo certbot certonly --webroot -w /var/www/example -d example.com
```

在这个例子中，我们正在尝试为 `example.com` 域名生成证书，该域名使用 `/var/www/example` 中的目录作为静态文件根目录。如果 certbot 能够验证域名，它将把证书文件写为 `/etc/letsencrypt/live/example.com/fullchain.pem`，将私钥写为 `/etc/letsencrypt/live/example.com/privkey.pem`，这些证书在 90 天内有效。

要使用这个新获取的证书，你可以输入上面提到的两个文件名来代替我们之前使用的自签名文件，这应该适用于上述任何配置。当然，你还需要保证验证的域名能正常访问到你的应用程序，因为这是浏览器接受证书有效的唯一方式。

如果你使用 nginx 作为反向代理，则可以在配置中创建映射，为 certbot 提供一个私有目录，以便在其中编写其验证文件。在下面的示例中，我扩展了上一节中显示的 HTTP 服务器块，以将所有 Let's Encrypt 的相关请求发送到你选择的特定目录：

```bash
server {
    listen 80;
    server_name example.com;
    location ~ /.well-known {
        root /path/to/letsencrypt/verification/directory;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}
```

当需要续订证书时，也会使用 Certbot。为此，你只需执行以下命令：

```bash
$ sudo certbot renew
```

如果系统中有任何证书即将过期，则执行上述命令会更新它们，并确保新证书保留在相同位置。如果希望获取续订的证书，则可能需要重新启动 Web 服务器。

## 获得 SSL A+认证

如果你使用 Let's Encrypt 的证书或其他已知 CA 的证书，并且你在此服务器上运行最近维护的操作系统，那么就 SSL 安全性而言，你可能已经拥有了很接近高评分的服务器。您可以前往[Qualys SSL Labs](https://www.ssllabs.com/ssltest)网站并获取报告，了解你服务器的排名。

接下来还有一点小事需要做，该报告将指出你需要改进的地方，但总的来说，我希望你会被告知服务器为加密通信公开的选项太宽泛或太弱，让你对已知漏洞持开放态度。

相对容易进行改进的一个地方是如何生成在加密密钥交换期间使用的系数，其通常具有相当弱的默认值。特别是，[Diffie-Hellman](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)系数需要相当长的时间才能生成，因此默认情况下服务器使用较小的数字来节省时间。但是我们可以预先生成强系数并将它们存储在一个文件中，然后 nginx 就可以使用它。使用 openssl 工具，你可以运行以下命令：

```bash
openssl dhparam -out /path/to/dhparam.pem 2048
```

如果你想要更强的系数，你可以改变 2048 或更大，如 4096。此命令需要一段时间才能运行，特别是如果你的服务器 CPU 数量不多的话，但是当它完成后，你就会拥有一个具有强系数的 `dhparam.pem`文件，你可以将其插入到 nginx 的 ssl 服务器模块中：

```bash
	ssl_dhparam /path/to/dhparam.pem;
```

接下来，你可能需要配置服务器允许加密通信的加密方式。这是我服务器上的加密列表：

```
	ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:!DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
```

在这个列表中，禁用的密码以 `!` 为前缀。SSL 报告将告诉你是否存在不推荐的密码。你必须不时地关注是否发现了需要修改这个加密列表的新漏洞。

你可以在下面找到我当前的 nginx SSL 配置，其中包括上述设置，以及我为解决 SSL 报告中的警告而添加的一些设置：

```bash
server {
    listen 443 ssl;
    server_name example.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    ssl_dhparam /path/to/dhparam.pem;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:!DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;
    add_header Strict-Transport-Security max-age=15768000;
    # ...
}
```

在文章开头，你可以看到我为我的网站获得的评分结果。如果你在所有类别中都达到 100％标记，则必须为配置添加更多的限制，但这也将限制可以连接到你的站点的客户端数量。通常，较旧的浏览器和 HTTP 客户端使用的是不是很强的加密方式，但如果禁用这些加密方式，则这些客户端将无法连接。（译者注：安全性与兼容性的选择，作为服务提供方应该考虑这一点，适当的降低服务的限制，来兼容较旧的请求方式，也是一种妥协。）因此，你基本上需要妥协，并且还需要定期查看安全报告，并随着时间的推移进行更新修正。

遗憾的是，对于这些最后的 SSL 改进的复杂程度，你必须使用专业级 Web 服务器，那么如果你不想使用 nginx，要找到一个支持这些设置的 web 服务，可选择的数量目前是很少的。我知道 Apache 支持，但除此之外，我不知道谁家还支持。

> 原文：[Running Your Flask Application Over HTTPS --- miguelgrinberg](https://blog.miguelgrinberg.com/post/running-your-flask-application-over-https)

--- EOF ---
