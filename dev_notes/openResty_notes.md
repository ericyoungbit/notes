
# penResty 

## 字符串处理优化

 - string.gsub / string.match 的方法无法jit 优化，尽量使用 ngx_lua 模块提供的 ngx.re.match 等接口
 - openresty 正则表达式 : https://moonbingbing.gitbooks.io/openresty-best-practices/lua/re.html


## luajit 2.1 新增 table.new / table.clear

 - 前者主要用来预分配 Lua table 空间，后者主要用来高效的释放 table 空间，并且它们都是可以被 JIT 编译的
 - 具体可以参考一下 OpenResty 捆绑的 lua-resty-\* 库

## Datatime

 - 在 Lua 中，函数 time、date 和 difftime 提供了所有的日期和时间功能。
    - 这些函数通常会引发不止一个昂贵的系统调用，同时无法为 LuaJIT JIT 编译，对性能造成较大影响。
 - 推荐使用 ngx_lua 模块提供的带缓存的时间接口
    - ngx.today, ngx.time, ngx.utctime, ngx.localtime, ngx.now, ngx.http_time，以及 ngx.cookie_time 等。

 
## Nginx Location 中避免使用 if

 - if 指令是 rewrite 模块中的一部分, 是实时生效的指令
 - 在 if 里写入一些非 rewrite 指令, 会出现不可预期的问题
 - 唯一正确的修复方式是完全禁用 if 中的非 rewrite 指令

## Nginx 静态文件服务

 - https://moonbingbing.gitbooks.io/openresty-best-practices/ngx/static_file.html

## 使用 Nginx 内置绑定变量

 - https://moonbingbing.gitbooks.io/openresty-best-practices/openresty/inline_var.html
 - `ngx.var.VARIABLE`

## location 重定向

 - 外部重定向:  ngx.redirect
 - 内部重定向:  ngx.exec

## 子查询
    
 - 发起非阻塞的内部请求
 - `ngx.location.capture`

## 不同阶段共享变量

 - 在 OpenResty 的体系中，可以通过共享内存的方式完成不同工作进程的数据共享
    - 可以通过 Lua 模块方式完成单个进程内不同请求的数据共享
    - 单个请求内不同阶段的数据共享使用  ngx.ctx 
 - ngx.ctx 是一个表
    - 它用来存储基于请求的 Lua 环境数据，其生存周期与当前请求相同 (类似 Nginx 变量)。
    - 每个请求，包括子请求，都有一份自己的 ngx.ctx 表
 - https://moonbingbing.gitbooks.io/openresty-best-practices/openresty/share_var.html


## 防止 SQL 注入

 - https://moonbingbing.gitbooks.io/openresty-best-practices/openresty/safe_sql.html
 - 对于 MySQL ，可以调用 ndk.set_var.set_quote_sql_str ，对用户提交的内容 进行一次过滤即可。

## 如何发起新 HTTP 请求

 - 利用 proxy_pass
 - 利用 cosocket

# LuaNginxModule

## 正确的记录日志

 - 日志 统一放到 log_by_lua\* 阶段
 - 配合 ngx.ctx

## OpenResty 的缓存

 - 使用 Lua shared dict
    - 所有 worker 之间共享的，内部使用的 LRU 算法
    - shared.dict 使用的是共享内存，每次操作都是全局锁
 - 使用Lua LRU cache
    - worker 级别的，不会在 Nginx wokers 之间共享
    - 永远不会触发锁，效率上有优势，并且没有 shared.dict 的体积限制，内存上也更弹性
    - 不同 worker 之间数据不同享，同一缓存数据可能被冗余存储。
 
## Sleep

 - ngx.sleep(0.1)

## 定时任务

 - https://moonbingbing.gitbooks.io/openresty-best-practices/ngx_lua/timer.html
 - ngx.timer.at

## 请求返回后继续执行

 - https://moonbingbing.gitbooks.io/openresty-best-practices/ngx_lua/continue_after_eof.html

## 动态限速

 - limit_rate 限制响应速度
    - Nginx 有一个 $limit_rate，这个变量反映的是当前请求每秒能响应的字节数
 - limit_conn 限制连接数
 - limit_req 限制请求数

## 如何引用第三方 resty 库

 - OpenResty 引用第三方 resty 库非常简单，只需要将相应的文件拷贝到 resty 目录下即可。

## 获取work id

 - seq_id = ngx.worker.id()

## 缓存失效风暴

 - https://moonbingbing.gitbooks.io/openresty-best-practices/lock/cache-miss-storm.html

# Web 服务

## 连接池用法

 - https://moonbingbing.gitbooks.io/openresty-best-practices/web/conn_pool.html

## TIME_WAIT 问题

 - https://moonbingbing.gitbooks.io/openresty-best-practices/web/time_wait.html
 - 主动关闭的一方会进入 TIME_WAIT 状态，并且在此状态停留两倍的 MSL 时长。
 - Nginx 在某些情况下，会主动关闭客户端的请求，
    -  问题出在 User-Agent，Nginx 认为终端的浏览器版本太低，不支持 keep alive，所以直接 close 了。
 - `keepalive_disable none;`

