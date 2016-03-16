# Nginx服务器的代理服务
## 正向代理
作用域： http块、server块、location块  

- resolver 指令  
该指令用于指定DNS服务器的IP地址  
语法： `resolver address ... [valid=time];`  
 + *address*, DNS服务器的IP地址。如果不指定端口号，默认使用53端口。  
 + *time*， 设置数据包在网络中的有效时间。  
例：  
`resolver 127.0.0.1 [::1]:5353 valid=30s;` 
 
- resolver_timeout指令  
该指令用于设置DNS服务器域名解析超时时间。  
语法： `resolver_time time;`

- proxy_pass指令  
该指令用于设置代理服务器的协议和地址  

Tips: 在server块中，不要出现server_name指令，即不要设置虚拟主机的名称或IP。  

## 反向代理

### 反向代理的基本设置的21个指令
作用域： http块、server块、location块  

- 1.proxy_pass指令  
该指令用来设置被代理服务器的地址，可以是主机名称、IP地址加端口号的形式。  
语法： `proxy_pass URL;`  
 + URL为要设置的被代理服务器的地址，包含传输协议、主机名称或IP地址加端口号、URI等要素。传输协议通过是“http”或者“https”。指令同时还接受以“unix”开始的UNIX-domain套接字路径。  
例：  
```
proxy_pass http://www.myweb.name/uri;
proxy_pass https://www.myweb.name/uri;  
proxy_pass http://unix:/tmp/backend.socket:/uri;
```
　　如果被代理服务器是一组服务器的话，可以使用upstream指令配置后端服务器组。
例：  
```
upstream proxy_svrs
{
    server http://192.168.1.1:8001/uri/;
    server http://192.168.1.2:8001/uri/;
    server http://192.168.1.3:8001/uri/;
}
server 
{
    ...
    listen 80;
    server_name www.myweb.name;
    location /
    {
        proxy_pass proxy_svrs;
    }
}
```
　　在上例中，在组内的各个服务器中都指明了传输协议“http：//”，而在proxy_pass指令中就不需要指明了。如果将pstream指令的配置改为：  
```
upstream proxy_svrs
{
    server 192.168.1.1:8001/uri/;
    server 192.168.1.2:8001/uri/;
    server 192.168.1.3:8001/uri/;
}
```
　　我们就需要在proxy_pass指令中指明传输协议“http://”:  
`proxy_pass http://proxy_svrs;`  
　　URL中是否包含有URI，Nginx服务器的处理方式是不同的。如果URL中不包含URI，Nginx服务器不会改变原地址的URI；但是如果包含了URI，Nginx服务器将会使用新的URI替代原来的URI。  
　　例如下面的Nginx配置片段：
```
server
{
    ...
    listen 80;
    server_name www.myweb.name;
    location /server/
    {
        ...
        proxy_pass http://192.168.1.1;
    }
}
```
　　如果客户端使用“http://www.myweb.name ”发起请求，由于proxy_pass指令的URL变量不含有URI，所以转向的地址为“http://192.168.1.1/server ”。  
　　再来看看下面的Nginx配置片段：
```
servser 
{
	...
	listen 80;
	servet_name www.myweb.name;
	location /server/
	{
		...
		proxy_pass http://192.168.1.1/loc/;
	}
}
```
　　在该配置实例中，proxy_pass指令的URL包含了URL“/loc”。如果客户端仍然使用“http://www.myweb.name/server ”发起请求，Nginx服务器将会把地址转向“http://192.168.1.1/loc/ ”  

- 2.proxy_hide_header指令  
该指令用于设置Nginx服务器在发送HTTP响应时，隐藏一些头域信息。  
语法：  `proxy_hide_header field;`  
 + field为需要隐藏的头域。  

- 3.proxy_pass_header指令  
默认情况下，Nginx服务器在发送响应报文时，报文头中不包含“Date”、“Server”、“X-Accel”等来自被代理服务器的头域信息。该指令可以设置这些头域信息以被发送。  
语法：  `proxy_pass_herder field;`  
 + fidle为需要发送的头域。

- 4.proxy_pass_request_body指令  
该指令用于配置是否将客户端请求的请求体发送给代理服务器。  
语法：  `proxy_pass_request_body on | off`  
默认设置为开启（on）。

- 5.proxy_pass_request_headers指令  
该指令用于配置是否将客户端请求的请求头发送给代理服务器。  
语法：  `proxy_pass_request_headers on | off;`  
默认设置为开启（on）。

- 6.proxy_set_header指令
该指令可以更改Nginx服务器接收到的客户端请求的请求头信息，然后将新的请求头发送给被代理的服务器。  
语法：  `proxy_set_header field value;`  
 + field，要更改的信息所在的头域。
 + value，更改的值，支持使用文本、变量或者变量的组合。  
默认情况下，该指令的设置为：
```
proxy_set_header Host $proxy_host;
proxy_set_header Connection close;
```

- 7.proxy_set_body指令  
该指令可以更改Nginx服务器接收到的客户端请求的请求体信息，然后将新的请求体发送给被代理的服务器。  
语法：  `proxy_set_body value;`  
 + value， 要更改的信息，支持使用文本、变量或者变量的组合。

- 8.proxy_bind指令  
强制将与代理主机的连接绑定到指定的IP地址。通俗来讲就是，在配置了多个基于名称或者基于IP的主机的情况下，如果我们希望代理连接由指定的主机处理，就可以使用该指令进行配置。  
语法：  `proxy_bind address;`  
 + address，指定主机的IP地址。

- 9.proxy_connect_timeout指令  
该指令配置Nginx服务器与后端被代理服务器尝试建立连接的超时时间。  
语法：  `proxy_connect_timeout time;`  
 + time，设置的超时时间，默认为60s。

- 10.proxy_read_timeout指令  
该指令配置Nginx服务器向后端被代理服务器（组）发出read请求后，等待响应的超时时间。  
语法：  `proxy_read_timeout time;`  
 + time，设置的超时时间，默认为60s。

- 11.proxy_send_timeout指令  
该指令配置Nginx服务器向后端被代理服务器（组）发出write请求后，等待响应的超时时间。  
语法：  `proxy_send_timeout time;`  
 + time，设置的超时时间，默认为60s。

- 12.proxy_http_version指令  
该指令用于设置Nginx服务器提供代理服务的HTTP协议版本。  
语法：  `proxy_http_version 1.0 | 1.1;`  
默认设置为1.0版本。

- 13.proxy_method指令  
该指令用于设置Nginx服务器请求被代理服务器时使用的请求方法，一般为POST或者GET。设置了该指令，客户端的请求方法将被忽略。  
语法：  `proxy_method method;`  
 + method，可以设置为POST或者GET，注意不加引号。
 
- 14.proxy_ignore_client_abort指令  
该指令用于设置在客户端中断网络请求时，Nginx服务器是否中断对被代理服务器的请求。  
语法：  `proxy_ignore_client_abort on | off;`  
默认设置为off，当客户端中断网络请求时，Nginx服务器中断对被代理服务器的请求。

- 15.proxy_ignore_headers指令  
该指令用于设置一些HTTP响应头中的头域，Nginx服务器接收到被代理服务器的响应数据后，不会处理被设置的头域。  
语法：  `proxy_ignore_headers field ...;`  
 + field，要设置的HTTP响应头的头域，例如“X-Accel-Redirect”、“X-Accel-Expires”、“Expires”、“Cache-Contorl”或“Set-Cookie”等。

- 16.proxy_redirect指令  
该指令用于修改被代理服务器返回的响应头中的“Location”头域和“Refresh”头域，与proy_pass指令配合使用。比如，Nginx服务器通过proxy_pass指令将客户端的请求地址重写为被代理服务器的地址，那么Nginx服务器返回给客户端的响应头中“Location”头域显示的地址就应该和客户端发起请求的地址相对应，而不是代理服务器直接返回的地址信息，否则就会出问题。  
语法：  
```
1 proxy_redirect redirect replacement;
2 proxy_redirect default;
3 proxy_redirect off;
```
 + redirect，匹配“Location”头域值的字符串，支持变量的使用和正则表达式。  
 + replacement，用于替换redirect变量内容的字符串，支持变量的使用。  
　　对于第1个结构，假设被代理服务器返回的响应头中“Location”头域为：  
`Location: http://localhost:8081/proxy/some/uri/`  
　　该指令设置为：  
`proxy_redirect http://localhost:8081/proxy/ http://myweb/frontend/;`  
　　Nginx服务器会将“Location”头域的信息更改为：  
`Location: http://myweb/frontend//some/uri/`  
　　结构２使用default，代表使用location块的uri变量作为replacement，并使用proxy_pass变量作为redirect。下面两段配置的效果是相同的。  
```
#配置1
location /server/
{
    proxy_pass http://proxyserver/source/;
    proxy_redirect default;
}
#配置2
location /server/
{
    proxy_pass http://proxyserver/source/;
    proxy_rediret http://proxyserver/source/ /server/;
}
```  
　　使用结构３可以将当前作用域下所有的proxy_redirect指令配置全部设置为无效。

- 17.proxy_intercept_errors指令  
该指令用于配置一个状态是开启还是关闭。在开启该状态时，如果被合理的服务器返回的HTTP状态码为400或者大于400，则Nginx服务器使用自己定义的错误页（使用error_pasge指令）；如果关闭该指令，Nginx服务器直接将被代理服务器返回的HTTP状态返回给客户端。  
语法： `proxy_intercept_errors on | off`  

- 18.proxy_headers_hash_max_size指令  
该指令用于配置存放HTTP报文头的哈希表的最大容量。  
语法： `proxy_headers_hash_max_size size;`  
 + size，HTTP报文头哈希表的容量上限，默认为512个字符。  

- 19.proxy_headers_hash_bucket_size指令  
该指令用于配置存放HTTP报文头的哈希表容量的单位大小。  
语法： `proxy_headers_hash_bucket_size size;`  
 + size，设置的容量，默认为64个字符。  

- 20.proxy_next_upstream指令  
在配置Nginx服务器反向代理功能时，如果使用upstream指令一组服务器作为被代理服务器，服务器组中各服务器的访问规则遵循upstream指令配置的轮循规则，同时可以使用该指令配置在发生哪些异常情况时，将请求顺次交由下一个组内服务器处理。  
语法： `proxy_next_upstream status ...;`  
 + status，设置的服务器返回状态，可以是一个或多个。这些状态包括：  
   - error，在建立连接、向被代理的服务器发送请求或者读取响应头时服务器发生连接错误。  
   - timeout，在建立连接、向被代理的服务器发送请求或者读取响应头时服务器发生连接超时。
   - invalid_header，被代理服务器返回的响应头为空或者无效。
   - http_500|http_502|http_503|http_504|http_404，被代理的服务器返回500、502、503、504或者404状态代码。
   - off，无法将请求发送给被代理的服务器。
   
- 21.proxy_ssl_session_reuse指令  
该指令用于配置是否使用基于SSL安全协议的会话连接（“https://”）被代理的服务器。  
语法： `proxy_ssl_session_reuser on | off;`  
　　默认为开启（on）状态。如果我们在错误日志中发现“SSL3_GET_FINISHED:digest check failed”的情况，可以将该指令配置为关闭“off”状态。  

### Proxy Buffer的配置的7个指令
- 1.proxy_buffering指令
该指令用于配置是否启用或者关闭Proxy Buffer。  
语法： `proxy_buffering on | off;`  
　　默认设置为开启状态。  
　　开启和关闭Proxy Buffer还可以通过在HTTP响应头部的“X-Accel-Buffering”头域设置“yes”或者“no”来实现，但Nginx配置中proxy_ignore_headers指令的设置可能导致该头域设置失效。  

- 2.proxy_buffers指令  
该指令用于配置接收一次被代理服务器响应数据的Proxy Buffer个数和每个Buffer的大小。
语法： `proxy_buffers number size;`  
 + number，Proxy Buffer的个数。
 + size，每个buffer的大小，一般设置为内存页的大小。根据平台的不同，可能为4KB或者8KB。  
 　　默认设置： `proxy_buffer 8 4k|8k;`  

- 3.proxy_buffer_size指令  
该指令用于配置从被代理服务器获取的第一部分响应数据的大小，该数据中一般包含了HTTP响应头，Nginx服务器通过它来获取响应数据和被代理服务器的一些必要信息。  
语法： `proxy_buffer_size size;`  
 + size，设置的缓存大小，默认设置为4KB或者8KB，保持与proxy_buffers指令中的size变量相同，当然也可以设置得更小。  

- 4.proxy_busy_buffers_size指令  
该指令用于限制处于BUSY状态的Proxy Buffer的总大小。  
语法： `proxy_busy_buffers_size size;`  
 + size，处于BUSY状态的缓存区总大小。默认设置为8KB或者16KB。  

- 5.proxy_temp_path指令  
该指令用于配置磁盘上的一个文件路径，该文件用于临时存放代理服务器的大体积响应数据。如果Proxy Buffer被装满后，响应数据仍然没有被Nginx服务器完全接收，响应数据就会被临时存放在该文件中。  
语法： `proxy_temp_path path [level1 [level2 [level3]]];`  
 + path，磁盘上存放临时文件的路径。  
 + levelN，在path变量设置的路径下第几级hash目录中存放临时文件。  

- 6.proxy_max_temp_file_size指令  
该指令用于配置所有临时文件的总体积大小，存放在磁盘上的临时文件大小不能超过该配置值，避免了响应数据过大造成磁盘空间不足的问题。  
语法： `proxy_max_temp_file_size size;`  
 + size，临时文件总体积上限值，默认设置为1024MB。  

- 7.proxy_temp_file_write_size指令  
该指令用于配置同时写入临时文件的数据量的总大小，合理的设置可以避免磁盘IO负载过重导致系统性能下降的问题。  
语法： `proxy_temp_file_write_size size;`  
 + size，数据量总大小上限值，默认设置根据平台的不同，可以为8KB或者16KB，一般与平台的内存页大小相同。  
