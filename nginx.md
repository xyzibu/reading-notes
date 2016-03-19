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
作用域： http块、server块、location块  

### 反向代理的基本设置的21个指令
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

### Proxy Cache的配置的12个指令  
　　Proxy Cache机制依赖于Proxy Buffer机制，只有在Proxy Buffer机制开启的情况下Proxy Cache的配置才发挥作用。  
- 1.proxy_cache指令  
该指令用于配置一块公用的内存区域的名称，该区域可以存放缓冲的索引数据。这些数据在Nginx服务器启动时由缓存索引重建进程负责建立，在Nginx服务器的整个运行过程中由缓存管理进程负责定时检查过期数据、检索等管理工作。  
语法： `proxy_cache zone | off;`  
 + zone，用于存放缓存索引的内存区域的名称。
 + off，关闭proxy_cache功能，是默认的设置。  
从Nginx 0.7.66开始，Proxy Cache机制开启后会检查被代理服务器响应数据HTTP头中的“Cache-Control”头域、“Expires”头域。当“Cache-Control”头域中的值为“no-cache”、“no-store”、“private”或者“max-age”赋值为0或无意义时，当“Expires”头域包含一个过期的时间时，该响应数据不被Nginx服务器缓存。这样做的主要目的是为了避免私有的数据被其他客户端得到。

- 2.proxy_cache_bypass指令  
该指令用于配置Nginx服务器向客户端发送响应数据时，不从缓存中获取的条件。这些条件支持使用Nginx配置的常用变量。  
语法： `proxy_cache_bypass sring ...;`  
 + string，条件变量，支持设置多个，当至少有一个字符串指令不为空或者不等于0时，响应数据不从缓存中获取。  
例：  
```
proxy_cache_bypass $cookie_nocache $arg_nocache $arg_comment  
$http_pragma $http_authorization;
```  

- 3.proxy_cache_key指令  
该指令用于设置Nginx服务器在内存中为缓存数据建立索引时使用的关键字。  
语法： `proxy_cache_key string;`  
 + string，设置的关键字，支持变量。  
在Nginx 0.7.48之前的版本中默认的设置为：  
`proxy_cache_key $scheme$proxy_host$request_uri;`  
如果我们希望缓存数据包含服务器主机名称等关键字，则可以将该指令设置为：  
`proxy_cache_key "$scheme$host$request_uri"'`  
在Nginx 0.7.48之后的版本中，通常使用以下配置：  
`proxy_cache_key $scheme$proxy_host$uri$is_args$args;`  

- 4.proxy_cache_lock指令  
该指令用于设置是否开启缓存的锁功用。在缓存中，某些数据项可以同时被多个请求返回的响应数据填充。开启该功能后，Nginx服务器同时只能有一个请求填充缓存中的某一数据项，这相当于给该数据上锁，不允许其他请求操作。其他的请求如果也想填充该项，必须等待该数据项的锁被释放。这个等待时间由porxy_cache_lock_timeout指令配置。  
语法： `proxy_cache_lock on | off;`  
　　默认情况下，设置为关闭状态。

- 5.proxy_cache_lock_timeout指令  
该指令用于设置缓存的锁功能开启以后锁的超时时间。  
语法： `proxy_cache_lock_timeout time;`  
 + time，设置的时间，默认为5s。  

- 6.proxy_cache_min_uses指令  
该指令用于设置客户端请求发送的次数，当客户端向被代理服务器发送相同请求达到该指令设定的次数后，Nginx服务器才能该请求的响应数据做缓存。合理设置该值可以有效地降低硬盘上缓存数据的数量，并提高缓存的命中率。  
语法： `proxy_cache_min_uses number;`  
 + number，设置的次数，默认设置为1。  

- 7.proxy_cache_path指令  
该指令用于设置Nginx服务器存储缓存数据的路径以及和缓存索引相关内容。  
语法： 
```proxy_cache_path [levels=levels] kesy_zone=name:size1 [inactive=time]  
[max_size=size2] [loader_files=number] [loader_sleep=time2] [loader_threshold=time3];```  
 + path，设置缓存数据存放的根路径，该路径应该是预先存在于磁盘上的。
 + levels，设置在相对于path指定目录的第几级hash目录中缓存数据。levels=1，表示一级hash目录；levels=1:2，表示两级，依次类推。目录的名称是基于请求URL通过哈希算法获取到的。  
 + name:size1，Nginx服务器的缓存索引重建进程在内存中为缓存数据建立索引，这一对变量用来设置存放缓存索引的内存区域的名称和大小。
 + time1，设置强制更新缓存数据的时间，当硬盘上的缓存数据在设定的时间内没有被访问时，Nginx服务器就强制从硬盘上将其删除，下次客户端访问该数据时重新缓存。该指令默认设置为10s。
 + size2，设置硬盘中缓存数据的大小限制。硬盘中的缓存源数据由Nginx服务器的缓存管理进程进行管理，当缓存的大小超过该变量设置时，缓存管理进程将根据最近最少被访问的策略删除缓存。
 + number，设置缓存索引重建进程每次加载的数据元素的数量上限。在重建缓存索引的过程中，进程通过一系列的递归遍历读取硬盘上的缓存数据目录及缓存数据文件，对每个数据文件中的缓存数据在内存中建立对应的索引，我们称每建立一个索引为加载一个数据元素。进程在每次遍历过程中可以同时加载多个数据元素，该值限制了每次遍历中同时加载的数据元素的数量。默认设置为100。
 + time2，设置缓存索引重建进程在一次遍历结束、下次遍历开始之间的暂停时长。默认设置为50ms。
 + time3，设置遍历一次磁盘缓存源数据的时间上限。默认设置为200ms。　　
例：  
```
proxy_cache_path /nginx/cache/a levels=1 keys_zone=a:10m;
proxy_cache_path /nginx/cache/b levels=2:2 keys_zone=b:100m;
proxy_cache_path /nginx/cache/c levels=1:1:2 keys_zone=c:1000m;
```
**注意**  
该指令只能放在http块中。  

- 8.proxy_cache_use_stale指令  
如果Nginx在访问被代理服务器过程中出现被代理的服务器无法访问或者访问错误的现象时，Nginx服务器可以使用历史缓存响应客户端的请求，这些数据不一定和被代理服务器上最新的数据相一致，但对于更新频率不高的后端服务来说，Nginx服务器的该功能在一定程度上能够为客户端提供不间断访问。该指令用来设置一些状态，当后端被代理的服务器处于这些状态时，Nginx服务器启用该功能。  
语法：  
```
proxy_cache_use_stale error | timeout | invalid_header | updating | http_500 | http_502 
| http_503 | http_504 | http_404 | off ...;
``` 
其中的updating状态并不是指被代理服务器在updating状态，而是指客户端请求的数据在Nginx服务器中正好处于更新状态。
该指令的默认设置为off。

- 9.proxy_cache_valid指令  
该指令可以针对不同的HTTP响应状态设置不同的缓存时间。  
语法： `proxy_cache_valid [ code ...] time;`  
 + code，设置HTTP响应的状态码。该指令可选，如果不设置Nginx服务器只为HTTP状态代码为200、301和302的响应数据做缓存。可以使用“any”表示缓存所有该指令中未设置的其他响应数据。
 + time，设置缓存时间。  
例：  
```
proxy_cache_valid 200 302 10m;
proxy_cache_valid 301 1h;
proxy_cache_vali any 1m;
```
该例子中，对返回状态为200和302的响应数据缓存10分钟，对返回状态为301的响应数据缓存1小时，对返回状态为非200、301和302的响应数据缓存1分钟。

- 10.proxy_no_cache指令  
该指令用于配置在什么情况下不使用cache功能。  
语法： `proxy_no_cache string ...;`  
 + string，可以是一个或多个变量。当string的值不为空或者不为“0”时，不启用cache功能。

- 11.proxy_store指令  
该指令配置是否在本地磁盘缓存来自被代理服务器的响应数据。这是Nginx服务器提供的另一种缓存数据的方法，但是该功能相对Proxy Cache简单一些，它不提供缓存过期更新、内存索引建立等功能，不占用内存空间，对静态数据的效果比较好。  
语法：` proxy_store on | off | string;`
 + on | off，设置是否开启Proxy Store功能。如果使用变量on，功能开启，缓存文件会存放到alias指令或root指令设置的本地路径下。默认设置为off。
 + string，自定义缓存文件的存放路径。如果使用变量string，Proxy Stote功能开启，缓存文件会存放到指定的本地路径下。
Proxy Stroe方法多使用在被代理服务器端发生错误的情况下，用来缓存被代理服务器的响应数据。

- 12.proxy_store_access指令  
该指令用于设置用户或用户组地Proxy Store缓存的数据的访问权限。  
语法： `proxy_store_access users:permissions ...;`  
 + users，可以设置为user、group或者all。
 + permissions，设置权限。  
例：  
```
location /images/
{
    root /data/www;
    error_page 404 = /fech$uri;     #定义了404错误的请求页面
}
location /fetch/
{
    proxy_pass http://backend;
    proxy_stroe on;                 #开启Proxy Stroe方法
    proxy_stroe_access user:rw group:rw all:r;
    root /data/www;
}
```

# Nginx服务器的缓存机制  
## 基于memcached的缓存机制的6个指令  
在Nginx服务器的标准HTTP模块中有一个ngx_http_memcached_module模块，专门用于处理和memcached相关的配置和功能实现。

- 1.memcached_pass指令  
该指令用于配置memcached服务器的地址。  
语法： `memcached_pass address;`  
 + address，memcached服务器的地址，支持IP+端口的地址或者是域名地址。也可以使用upstream指令配置一个memcached服务器组，然后将address配置为upstream的名称。  

- 2.memcached_connect_timeout指令  
该指令用于配置连接memcached服务器的超时时间。  
语法： `memcached_connect_timeout time;`  
 + time，设置的超时时间，默认为60s。建议该时间不要超过75s。  

- 3.memcached_read_timeout指令  
该指令配置Nginx服务器向memcached服务器发出两次read请求之间的等待超时时间，如果在该时间内没有进行数据传输，连接将会被关闭。  
语法： `memcached_read_timeout time;`  
 + time，设置的超时时间，默认为60s。  

- 4.memcached_send_timeout指令  
该指令配置Nginx服务器向memcached服务器发出两次write请求之间的等待超时时间，如果在该时间内没有进行数据传输，连接将会被关闭。  
语法： `memcached_send_timeout time;`  
 + time，设置的超时时间，默认为60s。  

- 5.memcached_buffer_size指令  
该指令用于配置Nginx服务器用于接收memcached服务器响应数据的缓存区大小。  
语法： `memcached_buffer_size size;`  
 + size，设置的缓存区大小，一般是所在平台的内存页大小的倍数。默认设置为：  
 `memcached_buffer_size 4k|8k;`  

- 6.memcached_next_upstream指令  
该指令在配置了一组memcached服务器的情况下使用。服务器组中各memcached服务器的 访问规则遵循upstream指令配置的轮询规则，同时可以使用该指令配置在发生哪些异常情况时，将请求顺次交由下一个组内服务器处理。  
语法： `memcached_next_upstream status ...;`  
 + status，设置的memcached服务器返回状态，可以是一个或者多个。这些状态包括：  
   - error，在建立连接、向memcached服务器发送请求或者读取时服务器发生连接错误。
   - timeout，在建立连接、向memcached服务器发送请求或者读取时服务器发生连接超时。
   - invalid_header，memcached服务器返回的响应头为空或者无效。
   - not_found，memcached服务器未找到对应的键/值对。
   - off，无法将请求发送给memcached服务器。
 

# Nginx服务器的邮件服务
## Nginx邮件服务的配置的12个指令
- 1.listen指令
