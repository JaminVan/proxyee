### HTTP 代理服务器

    支持HTTP、HTTPS、WebSocket,HTTPS采用动态签发SSL证书,可以拦截http、https的报文并进行处理。
    例如：http(s)协议抓包,http(s)动态替换请求内容或响应内容等等。

#### HTTPS 支持

    需要导入项目中的CA证书(src/resources/ca.crt)至受信任的根证书颁发机构。
    可以使用CertDownIntercept拦截器，开启网页下载证书功能，访问http://serverIP:serverPort即可进入。
    
    注1：安卓手机上安装证书若弹出键入凭据存储的密码，输入锁屏密码即可。
    
    注2：由于android高版本新安全机制，系统不再信任用户安装的证书，你需要root后，使用
    cat ca.crt > $(openssl x509 -inform PEM -subject_hash_old -in ca.crt  | head -1).0
    命令生成 d1488b25.0 文件，然后把文件移动到
    /system/etc/security/cacerts/
    并给与644权限
    
    注3：在Android 7以上，即使你把证书添加进系统证书里，这个证书在chrome里也是不工作的。原因是chrome从2018年开始只信任有效期少于27个月的证书(https://www.entrustdatacard.com/blog/2018/february/chrome-requires-ct-after-april-2018)。所以你需要自行生成证书文件。

#### 生成证书文件
	key的生成
	openssl genrsa -out server.key 2048
	这样是生成RSA密钥，openssl格式，2048位强度。server.key是密钥文件名。
	
	key的转换
	openssl rsa -in server.key -out server_private.der -outform der
	把生成的key文件转成der文件
	
	csr的生成
	openssl req -new -key server.key -out server.csr
	需要依次输入国家，地区，组织，email。最重要的是有一个common name，可以写你的名字或者域名。
	
	crt的生成
	openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
	
	然后把server.crt 和 server_private.der 复制到项目的src/resources/中，然后更改com.github.monkeywie.proxyee.server.HttpProxyServer.java 中的第60和第61行

#### 二级代理

    可设置二级代理服务器,支持http,socks4,socks5。

#### 启动

```
//启动一个普通的http代理服务器，不解密https
new HttpProxyServer().start(9999);
```

```
//启动一个解密https的代理服务器，并且拦截百度首页，注入js和修改响应头
//当开启了https解密时，需要安装CA证书(`src/resources/ca.crt`)至受信任的根证书颁发机构。
HttpProxyServerConfig config =  new HttpProxyServerConfig();
config.setHandleSsl(true);
new HttpProxyServer()
    .serverConfig(config)
    .proxyInterceptInitializer(new HttpProxyInterceptInitializer() {
      @Override
      public void init(HttpProxyInterceptPipeline pipeline) {
        pipeline.addLast(new FullResponseIntercept() {

          @Override
          public boolean match(HttpRequest httpRequest, HttpResponse httpResponse, HttpProxyInterceptPipeline pipeline) {
            //在匹配到百度首页时插入js
            return HttpUtil.checkUrl(pipeline.getHttpRequest(), "^www.baidu.com$")
                && isHtml(httpRequest, httpResponse);
          }

          @Override
          public void handelResponse(HttpRequest httpRequest, FullHttpResponse httpResponse, HttpProxyInterceptPipeline pipeline) {
            //打印原始响应信息
            System.out.println(httpResponse.toString());
            System.out.println(httpResponse.content().toString(Charset.defaultCharset()));
            //修改响应头和响应体
            httpResponse.headers().set("handel", "edit head");
            /*int index = ByteUtil.findText(httpResponse.content(), "<head>");
            ByteUtil.insertText(httpResponse.content(), index, "<script>alert(1)</script>");*/
            httpResponse.content().writeBytes("<script>alert('hello proxyee')</script>".getBytes());
          }
        });
      }
    })
    .start(9999);
```

更多 demo 代码在 test 包内可以找到，这里就不一一展示了





#### 流程

SSL 握手
![SSL握手](https://raw.githubusercontent.com/monkeyWie/pic-bed/master/proxyee/20190918134332.png)

HTTP 通讯

![HTTP通讯](https://raw.githubusercontent.com/monkeyWie/pic-bed/master/proxyee/20190918134232.png)

#### 讲解
- [JAVA写HTTP代理服务器(一)-socket实现](https://segmentfault.com/a/1190000011810997)
- [JAVA写HTTP代理服务器(二)-netty实现](https://segmentfault.com/a/1190000011811082)
- [JAVA写HTTP代理服务器(三)-https明文捕获](https://segmentfault.com/a/1190000011811150)

#### 感谢
[![intellij-idea](idea.svg)](https://www.jetbrains.com/?from=proxyee)
