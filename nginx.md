# Nginx服务器的代理服务
## 正向代理
作用域： http块、server块、location块  

- resolver 指令  
该指令用于指定DNS服务器的IP地址  
语法： **`resolver address ... [valid=time];`**  
 + *address*, DNS服务器的IP地址。如果不指定端口号，默认使用53端口。  
 + *time*， 设置数据包在网络中的有效时间。  
例：  
`resolver 127.0.0.1 [::1]:5353 valid=30s;` 
 
- resolver_timeout指令  
该指令用于设置DNS服务器域名解析超时时间。  
语法： **`resolver_time time;`**

- proxy_pass指令  
该指令用于设置代理服务器的协议和地址  

Tips: 在server块中，不要出现server_name指令，即不要设置虚拟主机的名称或IP。  

## 反向代理
作用域： http块、server块、location块  

- proxy_pass指令  
该指令用来设置被代理服务器的地址，可以是主机名称、IP地址加端口号的形式。  
语法： **`proxy_pass URL;`**  
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
　　我们就需要在proxy_pass指令中指明传输协议“http：//”:  
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
　　如果客户端使用“http://www.myweb.name”发起请求，由于proxy_pass指令的URL变量不含有URI，所以转向的地址为“http://192.168.1.1/server”。  
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
　　在该配置实例中，proxy_pass指令的URL包含了URL“/loc”。如果客户端仍然使用“http://www.myweb.name/server”发起请求，Nginx服务器将会把地址转向“http://192.168.1.1/loc/”  

- proxy_hide_header指令
该指令用于设置Nginx服务器在发送HTTP响应时，隐藏一些头域信息。 
语法：  **`proxy_hide_header field;`**  
 + field为需要隐藏的头域。  

- proxy_pass_header指令
默认情况下，Nginx服务器在发送响应报文时，报文头中不包含“Date”、“Server”、“X-Accel”等来自被代理服务器的头域信息。该指令可以设置这些头域信息以被发送。
语法：  **`proxy_pass_herder field;`**  
 + fidle为需要发送的头域。

- proxy_pass_request_body指令
该指令用于配置是否将客户端请求的请求体发送给代理服务器。
语法：  **`proxy_pass_request_body on | off`**
默认设置为开启（on）。

- proxy_pass_request_headers指令
该指令用于配置是否将客户端请求的请求头发送给代理服务器。
语法：  **`proxy_pass_request_headers on | off;`**
默认设置为开启（on）。

- proxy_set_header指令
该指令可以更改Nginx服务器接收到的客户端请求的请求头信息，然后将新的请求头发送给被代理的服务器。
语法：  **`proxy_set_header field value;`**
 + field，要更改的信息所在的头域。
 + value，更改的值，支持使用文本、变量或者变量的组合。
默认情况下，该指令的设置为：
```
proxy_set_header Host $proxy_host;
proxy_set_header Connection close;
```

- proxy_set_body指令
该指令可以更改Nginx服务器接收到的客户端请求的请求体信息，然后将新的请求体发送给被代理的服务器。
语法：  **`proxy_set_body value;`**
 + value， 要更改的信息，支持使用文本、变量或者变量的组合。

- proxy_bind指令
强制将与代理主机的连接绑定到指定的IP地址。通俗来讲就是，在配置了多个基于名称或者基于IP的主机的情况下，如果我们希望代理连接由指定的主机处理，就可以使用该指令进行配置。
语法：  **`proxy_bind address;`**
 + address，指定主机的IP地址。

- proxy_connect_timeout指令
该指令配置Nginx服务器与后端被代理服务器尝试建立连接的超时时间。
语法：  **`proxy_connect_timeout time;`**
 + time，设置的超时时间，默认为60s。

- proxy_read_timeout指令
该指令配置Nginx服务器向后端被代理服务器（组）发出read请求后，等待响应的超时时间。
语法：  **`proxy_read_timeout time;`**
 + time，设置的超时时间，默认为60s。

- proxy_send_timeout指令
该指令配置Nginx服务器向后端被代理服务器（组）发出write请求后，等待响应的超时时间。
语法：  **`proxy_send_timeout time;`**
 + time，设置的超时时间，默认为60s。

- proxy_http_version指令
该指令用于设置Nginx服务器提供代理服务的HTTP协议版本。
语法：  **`proxy_http_version 1.0 | 1.1;`**
默认设置为1.0版本。

- proxy_method指令
该指令用于设置Nginx服务器请求被代理服务器时使用的请求方法，一般为POST或者GET。设置了该指令，客户端的请求方法将被忽略。
语法：  **`proxy_method method;`**
 + method，可以设置为POST或者GET，注意不加引号。
 
- proxy_ignore_client_abort指令
该指令用于设置在客户端中断网络请求时，Nginx服务器是否中断对被代理服务器的请求。
语法：  **`proxy_ignore_client_abort on | off;`**
默认设置为off，当客户端中断网络请求时，Nginx服务器中断对被代理服务器的请求。

- proxy_ignore_headers指令  
该指令用于设置一些HTTP响应头中的头域，Nginx服务器接收到被代理服务器的响应数据后，不会处理被设置的头域。
语法：  **`proxy_ignore_headers field ...;`**
 + field，要设置的HTTP响应头的头域，例如“X-Accel-Redirect”、“X-Accel-Expires”、“Expires”、“Cache-Contorl”或“Set-Cookie”等。

