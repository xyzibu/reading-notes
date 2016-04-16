# Nginx服务器的基础配置
## 用于调试进程和定位问题的配置项
- 1.是否以守护进程方式运行Nginx  
语法：`daemon on | off;`  
默认：`daemon on;`  
调试时可以关闭守护进程方式。

- 2.是否以master/worker方式工作  
语法：`master_process on | off;`  
默认：`master_process on;`  
如果用off关闭了master_process方式，就不会fork出worker子进程来处理请求，而是用master进程自身来处理请求。

- 3.是否处理几个特殊的调试点  
语法：`debug_points [stop | abort]`  
Nginx在一些关键的错误逻辑中设置了调试点。如果设置了debug_points为stop，那么Nginx的代码执行到这些调试点时就会发出SIGSTOP信号以用于调试。如果debug_points设置为abort，则会产生一个coredump文件，可以使用gdb来查看Nginx当时的各种信息。  
通常不会使用这个配置项。

- 4.仅对指定的客户端输出debug级别的日志  
语法：`debug_connection [IP | CIDR]`  
配置块：events块  
仅仅来处IP地址的请求才会输出debug级别的日志，其他请求仍然沿用error_log中配置的日志级别。  
此配置对修复Bug很有用，特别是定位高并发请求下才会发生的问题。

- 5.限制coredump核心转储文件的大小  
语法：`worker_rlimit_core size;`  

- 6.指定coredump文件生成目录  
语法：`working_directory path;`  
worker进程的工作目录。此配置项的唯一用途就是设置coredump文件所放置的目录，协助定位问题。因此，需要确保worker进程有权限向working_directory指定的目录中写入文件。

## 优化性能的配置项
- 1.配置允许生成的worker process数  
配置块：全局块  
语法：`worker_process number | auto;`  
 + number，指定Nginx进程最多可以产生的worker process数。
 + auto，设置此值，Nginx进程将会自动检测。  
默认的配置中，number=1。

- 2.绑定Nginx worker进程到指定的CPU内核  
语法：`worker_cpu_affinity cpumask [cpumask...]`  
例如，如果有4颗CPU内核，就可以进行如下配置：  
```
worker_process = 4;
worker_cpu_affinity 1000 0100 0010 0001;
```

- 3.SSL配件加速  
语法：`ssl_engine device;`  
如果服务器上有SSL硬件加速设备，那么就可以进行配置以加快SSL协议的处理速度。用户可以使用OpenSSL提供的命令来查看是否有SSL硬件加速设备：  
`openssl engine -t`

- 4.系统调用gettimeofday的执行频率  
语法：`timer_resolution t;`  
默认情况下，每次内核的事件调用（如epoll、select、poll、kqueue等）返回时，都会执行一次gettimeofday，实现用内核中的时钟来更新Nginx中的缓存时钟。在早期的Linux内核中，gettimeofday的执行代价不小，因为中间有一次内核态到用户态的内存复制。当需要降低gettimeofday的调用频率时，可以使用timer_resolution配置。  
但是在目前的大多数内核中，gettimeofday只是一次vsyscall，仅仅对共享内存页中的数据做访问，并不是通常的系统调用，代价并不大，一般不必使用这个配置。  

- 5.Nginx worker进程优先级设置  
语法：`worker_priority nice;`  
默认：`worker_priority 0;`  

## 事件类配置项
- 1.设置网络连接的序列化  
配置块：events块   
语法：`accept_mutex on | off;`  
默认：`accept_mutex on;`    
accept_mutex可以让多个worker进程轮流地、序列化地与新的客户端建立TCP连接。当某一个worker进程建立的连接数量达到worker_connections配置的最大连接数的7/8时，会大大地减小该worker进程试图建立新TCP连接的机会，以此实现所有worker进程之上处理的客户端请求数尽量接近。  
accept锁默认是开启的，如果关闭它，那么建立TCP连接的耗时会更短，但worker进程之间的负载会非常不均衡，因此不建议关闭它。    
   
- 2.lock文件的路径
语法：`lock_file path/file;`  
默认：`lock_file logs/nginx.lock;`  
accept锁可能需要这个lock文件，如果accpet锁关闭，lock_file配置完全不生效。如果打开了accept锁，并且由于编译程序、操作系统架构等因素导致Nginx不支持原子锁，这时才会使用文件锁实现accept锁，这样lock_file指定的lock文件才会生效。  

- 3.使用accept锁后到真正建立连接之间的延迟时间  
语法：`accept_mutex_delay Nms;`  
默认：`accept_mutex_delay 500ms;`  
在使用accept锁后，同一时间只有一个worker进程能够获取到accept锁。这个accept锁不是阻塞锁，如果取不到会立即返回。如果有一个worker进程试图取accept锁而没有取到，它至少要等accept_mutex_delay定义的时间间隔后才能再次试图获取锁。

- 4.设置是否允许同时连接多个网络 
配置块：events块  
语法：`multi_accept on | off;`  
此指令默认为关闭（off）指令，即每个worker process一次只能接收一个新到达的网络连接。

- 5.事件驱动模型的选择 
配置块：events块  
语法：`use method;`  
 + method，select、poll、kqueue、epoll、rtsig、/dev/poll、eventport

- 6.配置最大连接数   
配置块：events块  
语法：`worker_connections number;`  
用来设置允许每一个worker process同时开启的最大连接数。此指令的默认设置为512。  
**注意**  
这里的number不仅仅包括和前端用户建立的连接数，而是包括所有可能的连接数。另外，number值不能大于操作系统支持打开的最大文件句柄数。

## HTTp核心模块配置
### 虚拟主机与请求的分发
- 1.配置网络监听   
配置监听使用listen指令，其配置方法有三种。  
(1).配置监听的IP地址。  
语法：  
```
listen address[:port] [default_server] [setfib=number] [backlog=number] [rcvbuf=size] [sendbuf=size] [deferred] 
[accept_filter=filter] [bind] [ssl];
```  
(2).配置监听端口  
语法：
```
listen port [default_server] [setfib=number] [backlog=number] [rcvbuf=size] [sendbuf=size] [deferred] 
[accept_filter=filter] [bind] [ipv6only=on|off] [ssl];
```  
(3).配置UNIX Domain Socket  
语法：  
```
listen unix:path [default_server] [backlog=number] [rcvbuf=size] [sendbuf=size] [deferred] 
[accept_filter=filter] [bind] [ssl];
```  
 + address，IP地址，如果是IPv6的地址，需要使用中括号“[]”括起来，比如[fe80::1]等。  
 + port，端口号，如果只定义了IP地址没有定义端口号，就使用80端口。
 + path，socket文件路径。
 + default_server，标识符，将此虚拟主机设置为address:port的默认主机。
 + setfib=number，Nginx-0.8.44中使用这个变量为监听socket关联路由表，目前只对FreeBSD起作用，不常用。
 + backlog=number，设置监听函数listen()最多允许多少网络连接同时处于挂起状态，在FreeBSD中默认为-1，其他平台默认为511。
 + rcvbuf=size，设置监听socket接收缓冲区大小。
 + sndbuf=size，设置监听socket发送缓冲区大小。
 + deferred，标识符，将accept()设置为Deferred模式。
 + accept_filter=filter，设置监听端口对请求的过滤，被过滤的内容不能被接收和处理。本指令只在FreeBSD和NetBSD 5.0+平台有效。filter可以设置为dataready或httpready。
 + bind，标识符，使用独立的bind()处理此address:port。一般情况下，对于端口相同而IP地址不同的多个连接，Nginx服务器将只使用一个监听命令，并使用bind()处理端口相同的所有连接。
 + ssl，标识符，设置会话连接使用SSL模式进行，此标识符和Nginx服务器提供的HTTPS服务有关。
 
- 2.基于名称的虚拟主机配置   
语法：`server_name name;`  
 + name，主机名，可以只有一个名称，也可以由多个名称并列，之间用空格隔开。在name中可以使用通配符，但通配符只能用在由三段字符串组成的名称的首段或尾段，或者由两段字符串组成的名称的尾段。在name中还可以使用正则表达式，并使用“~”作为正则表达式字符串的开始标记。  
由于server_name指令支持使用通配符和正则表达式两种配置名称的方式，因此在包含有多个虚拟主机的配置文件中，可能会出现一个名称被多个虚拟主机的server_name匹配成功。那么，来自这个名称的请求到底要交给哪个虚拟主机处理呢？Nginx服务器做出以下支付宝：
a.对于匹配方式的不同，按照以下的优先级选择虚拟主机，排在前面的优先处理请求。
 + 1.准确匹配server_name  
 + 2.通配符在开始时匹配server_name成功
 + 3.通配符在结尾时匹配server_name成功
 + 4.正则表达式匹配server_name成功
b.在以上四种匹配方式中，如果server_name被处于同一优先级的匹配方式多次匹配成功，则首次匹配成功的虚拟主机处理请求。

- 3.server_names_hash_bucket_size  
语法：`server_names_hash_bucket_size size;`  
默认：`server_names_hash_bucket_size 32|64|128;`  
配置块：http、server、location  
为了提高快速寻找到相应server name的能力，Nginx使用散列表来存储server name。此配置设置了每个散列桶占用的内存大小。

- 4.server_names_hash_max_size  
语法：`server_names_hash_max_size size;`  
默认：`server_names_hash_max_size 512;`  
配置块：http、server、location  
此配置会影响散列表的冲突率。值越大，消耗的内存就越多，但散列key的冲突率则会降低，检索速度也更快。值越小，消耗的内存就越小，但散列key的冲突率可能增高。  

- 5.重定向主机名称的处理  
语法：`server_name_in_redirect on | off;`  
默认：`server_name_in_redirect on;`  
配置块：http、server、location  
该配置需要配置server_name使用。在使用on打开时，表示在重定向请求时会使用server_name里配置的每一个主机名代替原先请求中的Host头部，而使用off关闭时，表示在重定向请求时使用请求本身的Host头部。  

- 6.配置location块
语法：`location [ = | ~ | ~* | ^~ | @] /uri/ {...}`  
配置块：server    
 + uri，待匹配的请求字符串，可以是不含正则表达式的字符串，也可以是包含有正则表达式的字符串。
 + 方括号里的部分是可选项，用来改变请求字符串与uri的匹配方式。在不添加此选项时，Nginx服务器首先在server块的多个location块中搜索是否有标准uri和请求字符串匹配，如果有多个可以匹配，就记录匹配度最高的一个。然后服务器再用location块中的正则uri和请求字符串匹配，当第一个正则uri匹配成功，结束搜索，并使用这个location块处理此请求；如果正则匹配全部失效，就使用刚才记录的匹配度最高的location块处理此请求。  
   + “=”，用于标准uri前，要求请求字符串与uri严格匹配。如果已经匹配成功，就停止继续向下搜索并立即处理此请求。
   + “~”，用于表示uri包含正则表达式，并且区分大小写。
   + “~*”，用于表示uri包含正则表达式，并且不区分大小写。
   + “^~”，表示只要前半部分与uri参数匹配即可。
   + “@”，仅用于Nginx服务内部请求之间的重定向，带有@的location不直接处理用户请求。

### 文件路径的定义  
- 1.配置请求的根目录   
语法：`root path;`  
默认：`root html;`  
配置块：http、server、location、if  
 + path，Nginx服务器接收到请求以后查找资源的根目录路径。

- 2.以alias方式设置资源路径  
语法：｀alias path;`    
配置块：location  
alias也是用来设置文件资源路径的，它与root的不同点主要在于如何解读紧跟location后面的uri参数。例如，如果一个请求的URI是/conf/nginx.conf，用户想访问的文件在/usr/local/nginx/conf/nginx.conf，那么用alias来设置的话如下：  
```
location /conf {
    alias /usr/local/nginx/conf/;
}
```  
如果用root设置如下：  
```
location /conf {
    root /usr/local/nginx/;
}
```  
使用alias时，在URI向实际文件路径的映射过程中，已经把location后配置的/conf丢弃掉，而root则会根据完整的URI请求来映射，这也是alias只能放在location块中的原因。  
 
- 3.设置网站的默认首页  
index指令有两个作用：一是用户在发出请求访问网站时，请求地址可以不写首页名称；二是可以对一个请求，根据请求内容而设置不同的首页。  
语法：`index file ...;`
默认：`index index.html;`  
配置块：http、server、location    
 + file，可以包含多个文件名，其间使用空格分隔，也可以包含其他变量。 

- 4.设置网站的错误页面  
语法：`error_page code [code...] [= | =answer-code] uri | @named_location;`   
配置块：http、server、location、if  
当对于某个请求返回错误码时，如果匹配上了error_page中设置的code，则重定向到新的URI中。例如：  
```
error_page 502 503 504 /50x.html;
error_page 404 = @fetch;
```
虽然重定向了URI，但返回的HTTP错误码还是与原来的相同。用户可以通过“=”来更改返回的错误码，例如：  
```
error_page 404 =200 /empty.gif;
```
也可以不指定确切的返回错误码，而是由重定向后实际处理的真实结果来决定，这时，只要把“=”后面的错误码去掉即可，例如：  
```
error_page 404 = /empty.gif;
```
如果不想修改URI，只是想让这样的请求重定向到另一个location中进行处理，那么可以这样设置：
```
location / {
    error_page 404 @fallback;
}

location @fallback {
    proxy_pass http://backend;
}
```

- 5.是否允许递归使用error_page  
语法：`recursive_error_page [on | off];`  
默认：`recursive_error_page off;`  
配置块：http、server、location  

- 6.try_files  
语法：`try_files path1 [path2] uri;`  
配置块：server、location  
尝试按照顺序访问每一个path，如果可以有效的读取，就直接向用户返回这个path对应的文件结束请求，否则继续向下访问。如果所有的path都找不到有效的文件，就重定向到最后的参数uri上。

### 内存及磁盘资源的分配
- 1.HTTP包体只存储到磁盘文件中  
语法：`client_body_in_file_only on | clean | off;`  
默认：`client_body_in_file_only off;`  
配置块：http、server、location  
当值为非off时，用户请求中的HTTP包体一律存储到磁盘文件中，即使只有0字节也会存储为文件。当请求结束时，如果配置为on，则这个文件不会被删除（该配置一般用于调试、定位问题），但如果配置为clean，则会删除该文件。  

- 2.HTTP包体尽量写入到一个内存buffer中  
语法：`client_body_in_single_buffer on | off;`  
默认：`client_body_in_single_buffer off;`  
配置块：http、server、location  
用户请求中的HTTP包体一律存储到内存buffer中。如果HTTP包体的大小超过了client_body_buffer_size设置的值，包体还是会 写入到磁盘文件中。  

- 3.存储HTTP头部的内存buffer大小  
语法：`client_header_buffer_size size;`  
默认：`client_header_buffer_size 1k;`  
配置块：http、server  
此配置定义了正常情况下Nginx接收用户请求中HTTP header部分（包括HTTP行和HTTP头部）时分配的内存buffer大小。有时，请求中的HTTP header部分可能会超过这个大小，这时large_client_header_buffers定义的buffer将会生效。

- 4.存储超大HTTP头部的内存buffer大小  
语法：`large_client_header_buffers number size;`  
默认：`large_client_header_buffers 4 8k;`  
配置块：http、server  
此配置定义了Nginx接收一个超大HTTP头部请求的buffer个数和每个buffer的大小。

- 5.存储HTTP包体的内存buffer大小  
语法：`client_body_buffer_size size;`  
默认：`client_body_buffer_size 8k/16k;`  
配置块：http、server、location  
此配置定义了Nginx接收HTTP包体的内存缓冲区大小。HTTP包体会先接收到指定的这块缓存中，之后才决定是否写入磁盘。  
**注意** 
如果用户请求中含有HTTP头部Content-Length，并且其标识的长度小于定义的buffer大小，那么Nginx会自动降低本次请求所使用的内存buffer，以降低内存消耗。  

- 6.HTTP包体的临时存放目录  
语法：`client_body_temp_path dir-path [ level1 [ level2 [ level3 ]]];`  
默认：`client_body_temp_path client_body_temp;`  
配置块：http、server、location  
在接收HTTP包体时，如果包体的大小大于client_body_buffer_size，则会以一个递增的整数命名并存放到client_body_temp_path指定的目录中。后面跟着的level1、level2、level3是为了防止一个目录下的文件数量太多，从而导致性能下降，因此使用了level参数，这样可以按照临时文件名最多再加三层目录，具体的值代表目录名的位数。例如：  
`client_body_temp_path /opt/nginx/client_temp 1 2;`  
如果新上传的HTTP包体使用123456作为临时文件名，就会被存放在这个目录中。  
`/opt/nginx/client_temp/6/45/123456`  

- 7.connection_pool_size  
语法：`connection_pool_size size;`  
默认：`connection_pool_size 256;`  
配置块：http、server  
Nginx对于每个建立成功的TCP连接会预先分配一个内存池，上面的size配置项将指定这个内存池的初始大小（即ngx_connection_t结构体中的pool内存池初始大小），用于减少内核对于小块内存的分配次数。需慎重设置，因为更大的size会使服务器消耗的内存增多，而更小的size则会引发更多的内存分配次数。

- 8.request_pool_size  
语法：`request_pool_size size;`  
默认：`request_pool_size 4k;`  
配置块：http、server  
Ngin开始处理HTTP请求时，将会为每个请求都分配一个内存池，size配置项将指定这个内存池的初始大小（即ngx_http_request_t结构中的pool内存池初始大小），用于减少内核对于小块内存的分配次数。TCP连接关闭时会销毁connection_pool_size指定的连接内存池，HTTP请求结束时会销毁request_pool_size指定的HTTP请求内存池，但它们的创建、销毁时间并不一致，因为一个TCP连接可能被复用于多个HTTP请求。
  
### 网络连接的设置  
- 1.读取HTTP头部的超时时间  
语法：`client_header_timeout time;`（默认单位：秒）  
默认：`client_header_timeout 60;`  
配置块：http、server、location  
客户端与服务器建立连接后将开始接收HTTP头部，在这个过程中，如果在一个时间间隔内没有读取到客户端发来的字节，则认为超时，并向客户端返回408（“Request timed out”）响应。  

- 2.读取HTTP包体的超时时间  
语法：`client_body_timeout time;`(默认单位：秒）  
默认：`client_body_timeout 60;`  
配置块：http、server、location  

- 3.发送响应的超时时间  
语法：`send_timeout time;`  
默认：`send_timeout 60;`  
配置块：http、server、location  
Nginx服务器向客户端发送了数据包，但客户端一直没有去接收这个数据包。如果某个连接超过send_timeout定义的超时时间，那么Nginx将会关闭这个连接。  

- 4.reset_timeout_connection  
语法：`reset_timeout_connection on | off;`  
默认：`reset_timeout_connection off;`  
配置块：http、server、location  
连接超时后将通过向客户端发送RST包来直接重置连接。这个选项打开后，Nginx会在某个连接超时后，不是使用正常情形下的四次握手关闭TCP连接，而是直接向客户发送RST重置包，不再等待用户的应答，直接释放Nginx服务器上关于这个套接字使用的所有缓存。相比正常的关闭方式，它使得服务器避免产生许多处于FIN_WAIT_1、FIN_WAIT_2、FIN_WAIT状态的TCP连接。  
使用RST重置包关闭连接会带来一些问题，默认情况下不会开启。  




## 正常运行的配置项
- 1.配置Nginx进程PID存放路径   
配置块：全局块  
语法：`pid file;`  
默认：`pid logs/nginx.pid;`  
 + file，指定存放路径和文件名称。可以是绝对路径，也可以是以Nginx安装目录为根目录的相对路径。  

- 2.配置运行Nginx服务器用户（组）  
配置块：全局块  
语法： `user user [group];`  
 + user，指定可以运行Nginx服务器的用户。
 + group，可选项，指定可以运行Nginx服务器的用户组。  
只有被设置的用户或者用户组才有权限启动Nginx进程，如果是其他用户尝试启动Nginx进程，将会报错。  
如果希望所有用户都可以启动Nginx进程，有两种方法：  
(1).将此指令行注释掉：  
`#user user [group];`  
(2).将用户（和用户组）设置为nobody：  
`user nobody nobody;`  
这也是user指令的默认配置。  

- 3.配置文件的引入   
配置块：任意地方  
语法：`include file;`  
 + file，要引入的配置文件，支持相对路径。

- 4.配置错误日志的存放路径    
配置块：全局块、http、server、location  
语法：`error_log file | stderr [debug | info | notice | warn | error | crit | alert | emerg];` 

- 5.更改一个worker进程的最大打开文件数限制  
语法：`worker_rlimit_nofile number;`  
如果没设置的话，这个值为操作系统的限制。设置后你的操作系统和Nginx可以处理比“ulimit -a”更多的文件，所以把这个值设高，这样Nginx就不会有“too many open files”问题了。

- 6.限制信号队列  
语法：`worker_rlimit_sigpending number;`  
设置每个用户发往Nginx的信号队列的大小。当某个用户的信号队列满了，这个用户再发送的信号量会被丢掉。


##　自定义服务日志
- access_log指令  
配置块：http、server、location  
语法：`access_log path [format [buffer=size]];`  
 + path，配置服务日志的文件存放的路径和名称。
 + format，可选项，自定义服务日志的格式字符串，也可以通过“格式串的名称”使用log_format指令定义好的格式。
 + size，配置临时存放日志的内存缓存区大小。  
默认配置：`access_log log/access.log combined;`  
其中，combined为log_format指令默认定义的日志格式字符串的名称。  
如果要取消记录服务日志的功能，则使用：`access_log off;`  

- log_format指令  
配置块：http  
语法：`log_format name string ...;`  
 + name，格式字符串的名称，默认的名字为combined。
 + string，服务日志的格式字符串。

## 配置允许sendfile方式传输文件
- sendfile指令  
配置块：http、server、location  
语法：`sendfile on | off;`  
默认值为off。  

- sendfile_max_chunk指令  
配置块：http、server、location  
语法：`sendfile_max_chunk size;`  
 + size，如果大于0，Nginx进程的每个worker process每次调用sendfile()传输的数据量最大不能超过这个值；如果设置为0，则无限制。默认值为0。
  
## 配置连接超时时间
- keepalive_timeout指令  
配置块：http、server、location  
语法：`keepalive_timeout timeout [header_timeout];`  
 + timeout，服务器端对连接的保持时间，默认值为75s。
 + header_timeout，可选项，在应答报文头的Keep-Alive域设置超时时间：“Keep-Alive:timeout=header_timeout”。
 
## 单连接请求数上限
- keepalive_requests指令  
配置块：server、location  
Nginx服务器端和用户端建立会话连接后，用户端通过此连接发送请求。此指令用于限制用户通过某一连接向Nginx服务器发送请求的次数。  
语法：`keepalive_request number;`  
默认设置为100。  
 

   

   


## 基于IP配置Nginx的访问权限
配置块：http、server、location
- allow指令
用于设置允许访问Nginx的客户端IP  
语法：`allow address | CIDR | all;`  
 + address，允许访问的客户端的IP，不支持同时设置多个。如果有多个IP需要设置，需要重复使用allow指令。
 + CIDR，允许访问的客户端的CIDR地址。
 + all，允许所有客户端访问。

- deny指令
用于设置禁止访问Nginx的客户端IP  
语法：`deny address | CIDR | all;`  
 + address，禁止访问的客户端的IP，不支持同时设置多个。如果有多个IP需要设置，需要重复使用deny指令。
 + CIDR，禁止访问的客户端的CIDR地址。
 + all，禁止所有客户端访问。

## 基于密码配置Nginx的访问权限
- auth_basic指令
用于开户或者关闭认证功能。  
语法：`auth_basic string | off;`  
 + string，开启认证功能，并配置认证时的指示信息。
 + off，关闭认证功能。

- auth_basic_user_file指令
设置包含用户名和密码信息的文件路径。  
语法：`auth_basic_user_file file;`  
 + file，密码文件的绝对路径。

# Nginx服务器的高级配置
## 针对IPv4的内核7个参数的配置优化
这里提及的参数是和IPv4网络有关的Linux内核参数。我们可以将这些内核参数的值追加到Linux系统的/etc/sysctl.conf文件中，然后使用如下命令使修改生效：
`#/sbin/sysctl -p`  
- 1.net.core.netdev_max_backlog参数
表示当每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许发送到队列的数据包的最大数目。一般默认值为128。Nginx服务器中定义的NGX_LISTEN_BACKLOG默认为511。可以设置为：
`net.core.netdev_max_backlog = 262144`

- 2.net.core.somaxconn参数
该参数用于调节系统同时发起的TCP连接数，一般默认值为128。在客户端存在高并发请求的情况下，该默认值较小，可能导致连接超时或者重传问题。可以设置为：  
`net.core.somaxconn = 262144`

- 3.net.ipv4.tcp_max_orphans参数
该参数用于设定系统中最多允许存在多少TCP套接字不被关联到任何一个用户文件句柄上。如果超过这个数字，没有与用户文件句柄关联的TCP套接字将立即被复位，同时给出警告信息。这个限制只是为了防止简单的DoS攻击。一般在系统内存比较充足的情况下，可以增大这个参数的赋值：  
`net.ipv4.tcp_max_orphans = 262144`

- 4.net.ipv4.tcp_max_syn_backlog参数
该参数用于记录尚未收到客户端确认信息的连接请求的最大值。对于拥有128MB内存的系统而言，此参数的默认值是1024，对小内存的系统则是128。一般在系统内存比较充足的情况下，可以增大这个参数的赋值：  
`net.ipv4.tcp_max_syn_backlog = 262144`

- 5.net.ipv4.tcp_timestamps参数
该参数用于设置时间戳，这可以避免序列号的卷绕。在一个1Gb/s的链路上，遇到以前用过的序列号的概率很大。当此值赋值为0时，禁用对于TCP时间戳的支持。在默认情况下，TCP协议会让内核接受这种“异常”的数据包。针对Nginx服务器来说，建议将其关闭：  
`net.ipv4.tcp_timestamps = 0`

- 6.net.ipv4.tcp_synack_retries参数
该参数用于设置内核放弃TCP连接之前向客户端发送SYN+ACK包的数量。为了建立对端的连接服务，服务器和客户端需要进行三次握手，第二次握手期间，内核需要发送一个SYN并附带一个回应前一个SYN的ACK，这个参数主要影响这个过程，一般赋值为1，即内核放弃连接之前发送一次SYN+ACK包，可以设置其为：  
`net.ipv4.tcp_synack_retries = 1`

- 7.net.ipv4.tcp_syn_retries参数
该参数的作用与上一个参数类似，设置内核放弃建立连接之前发送SYN包的数量，它的赋值和上个参数一样即可：  
`net.ipv4.tcp_syn_retries = 1`

## 针对CPU的Nginx配置代码的2个指令
- 1.worker_processes指令







# Nginx服务器的代理服务
## 正向代理
配置块： http、server、location  

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
配置块： http、server、location  

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
 + method，可以设置为POST或者GET，
 + 不加引号。
 
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
    + error，在建立连接、向被代理的服务器发送请求或者读取响应头时服务器发生连接错误。  
    + timeout，在建立连接、向被代理的服务器发送请求或者读取响应头时服务器发生连接超时。
    + invalid_header，被代理服务器返回的响应头为空或者无效。
    + http_500|http_502|http_503|http_504|http_404，被代理的服务器返回500、502、503、504或者404状态代码。
    + off，无法将请求发送给被代理的服务器。
   
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
    + error，在建立连接、向memcached服务器发送请求或者读取时服务器发生连接错误。
    + timeout，在建立连接、向memcached服务器发送请求或者读取时服务器发生连接超时。
    + invalid_header，memcached服务器返回的响应头为空或者无效。
    + not_found，memcached服务器未找到对应的键/值对。
    + off，无法将请求发送给memcached服务器。
 

# Nginx服务器的邮件服务
## Nginx邮件服务的配置的12个指令
- 1.listen指令  
配置块：server  
该指令用于配置邮件服务器监听的IP地址和端口。  
语法： `listen address:port;`
 + address，邮件服务器监听的IP地址，支持通配符“*”、主机名称。
 + port，邮件服务器监听端口。

- 2.server_name指令  
该指令用于为每个server块构成的虚拟主机配置的域名。  
语法： `server_name name;`  
 + name，配置的服务器域名。  
如果mail块中配置了多个虚拟主机，该指令只能在server块中配置；如果只有一个虚拟主机，该指令可以在mail块中配置。

- 3.protocol指令  
配置块：server  
该指令用于配置当前虚拟主机支持的协议。  
语法： `protocol imap | pop3 | smtp;`  

- 4.so_keepalive指令
配置块：mail、server  
该指令用于配置后端代理服务器是否启用“TCP keepalive”模式来处理Nginx邮件服务器转发的客户端连接。  
语法： `so_keepalive on | off;`  
默认情况下，该指令配置为off。

- 5.配置POP3协议
用于配置POP3协议的指令有两个：pop3_auth指令和pop3_capabilities指令。  
 + pop3_auth指令用于配置POP3认证用户的方式。  
配置块：mail、server  
语法： `pop3_auth method ...'`  
method支持以下配置：
     + plain，使用USER/PASS、AUTH PLAIN、AUTH LOGIN方法认证。这也是邮件服务器提供POP3协议支持时最基本的认证方式，也是Nginx邮件服务的默认配置。
     + apop，使用APOP方法认证。该方法需要客户端提供的密码是非加密密码。
     + cram-md5，使用AUTH CRAM-MD5方法认证。该方法也需要客户端提供的密码是非加密密码。
 + pop3_capabilities指令用于配置POP3协议的扩展功能。  
 配置块：mail、server  
语法： `pop3_capabilities extension ...;`  
     + extension，要加入POP3协议的扩展。  
默认配置：`pop3_capabilities TOP USER UIDL;`  

- 6.配置IMAP协议
用于配置IMAP协议的指令包括imap_auth指令、imap_capabilities指令和imap_client_buffer指令三个。前两个指令和配置POP3协议时使用的两个用法是相同的。  
 + imap_auth指令用于配置POP3认证用户的方式。
配置块：mail、server  
语法： `imap_auth method ...;`  
method支持以下配置：  
     + plain，使用AUTH=PLAIN方法认证。仍然是Nginx邮件服务提供IMAP协议的默认设置。
     + login，使用AUTH=LOGIN方法进行认证。
     + cram-md5，使用AUTH CRAM-MD5方法认证。该方法也需要客户端提供的密码是非加密密码。
 + imap_capabilities指令用于配置IMAP协议的扩展功能。
配置块：mail、server    
语法： `imap_capabilities extension ...;`  
     + extension，要加入IMAP协议的扩展。  
默认配置：`imap_capabilities IMAP4 IMAP4revl UIDPLUS;` 
 +  imap_client_buffer指令用于配置IMAP协议读数据缓存的大小。  
语法： `imap_client_buffer size;`  
     + size，配置的读缓存大小，一般为平台的一个内存页大小。  
默认配置：`imap_client_buffer 4K|8K;` 
 
- 7.配置SMTP协议
用于配置SMTP协议的指令包括smtp_auth指令和smtp_capabilities指令。它们的用法也和前面两个协议中的基本相同。  
 + smtp_auth指令用于配置SMTP认证用户的方式。  
配置块：mail、server  
语法： `smtp_auth method ...;`  
method支持以下配置：  
     + plain，使用AUTH=PLAIN方法认证。仍然是Nginx邮件服务提供IMAP协议的默认设置。
     + login，使用AUTH=LOGIN方法进行认证。
     + cram-md5，使用AUTH CRAM-MD5方法认证。该方法也需要客户端提供的密码是非加密密码。
默认配置：`smtp_auth plain;`  
 + smtp_capabilities指令用于配置SMTP协议的扩展功能。  
配置块：mail、server  
语法： `smtp_capabilities extension ...;`  
     + extension，要加入SMTP协议的扩展。

- 8.auth_http指令  
配置块：mail、server
用于配置Nginx提供邮件服务时的用于HTTP认证的服务器地址。  
语法： `auth_http URL;`  
 + URL，HTTP认证服务器的地址。  

- 9.auth_http_header指令  
配置块：mail、server  
通过该指令可以在Nginx服务器向HTTP认证服务器发起认证请求时，向请求头添加指定的头域。  
例：`auth_http_header X-Auth-Key "secret_string";`  

- 10.auth_http_timeout指令  
该指令用于配置Nginx服务器向HTTP认证服务器发起认证请求后等待响应的超时时间。  
语法： `auth_http_timeout time;`  
 + time，超时时间，默认60s。一般该时间设置不超过75s。

- 11.proxy_buffer指令  
配置块：mail、server  
该指令用于配置了后端代理服务器（组）的情况，用来设置Nginx服务器代理缓存的大小，一般为平台的一个内存页的大小。  
默认配置：`proxy_buffer 4k|8k;`  

- 12.proxy_pass_error_message指令  
该指令用于配置了后端代理服务器（组）的情况，用来配置是否将后端服务器上邮件服务认证过程中产生的错误信息发送给客户端。  
语法： `proxy_pass_error_message on | off;`  
默认情况下，该指令设置为off。  



  

