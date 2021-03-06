Varnish tuning
===

目前varnish在请求高峰期时，会返回504的错误。在查看日志时，发现只有请求上海的connection会返回504，其响应时间已到360s了。虽然很多时候504与抓数据有关，但varnish本身的调优也须注意。

参数修改
---

### thread\_pool\_max

在查看varnish配置时，发现connect\_timeout为默认的0.7s，thread\_pool\_max为500。查看varnishlog，发现当请求miss时，后端返回时间偶尔会达到5s。因此这里将connect\_timeout设置为6s。thread\_pool\_max设置为1500。

这里说下varnish的请求处理过程：varnish子程序负责对请求的处理，acceptor thread接收并代理新的connections，每个connection都是一个session，子程序的worker thread一对一的处理session。worker thread的最大个数由thread\_pool\_max和thread\_pools共同决定。这两个参数的最佳实践是thread\_pool\_max不超过5000，thread\_pools最好设置为2。子程序的Expiry thread将内存中过时的内容剔除掉。

那怎么判断threads数目是否足够呢？以下是参考：
> n_wrk, n_wrk_queued, n_wrk_drop: During normal operation, the n_wrk_queued counter should not grow. Once Varnish is out of threads, it will queue up requests and n_wrk_queued counts how many times this has happened. Once the queue is full, Varnish starts dropping requests without answering. n_wrk_drop counts how many times a request has been dropped. It should be 0.  
> n_lru_nuked: Counts the number of objects Varnish has had to evict from cache before they expired to make room for other content. If it is always 0, there is no point increasing the size of the cache since the cache isn’t full. If it’s climbing steadily a bigger cache could improve cache efficiency.

使用`varnishstat -f n_lru_nuked -n instancename `查看是否需要增加内存。

### http\_resp\_size

关于只有请求上海的connection出现问题，看了下shanghai返回的内容为32.7k，其他如北京，南京等均为25k左右，而http\_resp\_size默认值为32k，这里将其修改为64k。同时还修改了`sess_workspace`、`thread_pool_workspace`为128k。

需要注意的几点如下：
http\_resp\_size的内存是从worker workspace(参数thread\_pool\_workspace)中分配得到，thread\_pool\_workspace的默认大小为64k。http\_resp\_size设置的值需小于thread\_pool\_workspace的值。


关于workspace：

> Varnish uses workspaces to reduce the contention between each thread when they need to acquire or modify memory. There are multiple workspaces, but the most important one is the session workspace, which is used to manipulate session data. An example is changing www.example.com to example.com before it is entered into the cache, to reduce the number of duplicates.
>
> The session workspace is how much memory is allocated to each HTTP session for tasks like string manipulation of incoming headers. It is also used to modify the object returned from a web server before the precise size is allocated and the object is stored read-only.
>
> Some times you may have to increase the session workspace to avoid running out of workspace.
> 
> Remember: It’s all virtual - not physical memory.


修改VCL配置文件
---
### Ignore cache headers from the backend
修改vcl_fetch  
Some backends send headers that tell varnish not to cache elements. 

```
 # Cache-Control: no-cache, no-store, max-age=0, must-revalidate Pragma: no-cache
 sub vcl_fetch {
     if (beresp.ttl < 120s) {
         set beresp.ttl = 120s;
     }
 }
```

### grace mode
设置grace模式  

```
sub vcl_fetch {
  set beresp.grace = 5m;
}

sub vcl_recv {
    if (!req.backend.healthy) {
        set req.grace = 5m;
    } else {
        set req.grace = 15s;
    }
}
```

others
---
以下是需要注意的几点：

###1k的额外开销

> Varnish has an overhead on top of this for keeping track of the cache, so the actual memory footprint of Varnish will exceed what the ‘-s’ argument specifies if the cache is full. The current estimate (subject to change on individual Varnish-versions) is that about 1kB of overhead needed for each object. For 1 million objects, that means 1GB extra memory usage.
>
> In addition to the per-object overhead, there is also a fairly static overhead which you can calculate by starting Varnish without any objects. Typically around 100MB.


###some notes..

- connect_timeout  
当网络不好，最好将值设大点，但值太大的话，将导致varnish不能优雅的处理errors。

- thread\_queue\_limit  
 限定了每个thread\_pool的queue长度，当队列满了之后开始丢弃session。

- listen_depth  
 defines how many outstanding connections is allowed to queue up before the kernel starts dropping them.

Reference
---

- [varnish wiki](https://www.varnish-cache.org/trac/wiki)
- [grace and saint mode](https://www.varnish-software.com/static/book/Saving_a_request.html)
- [Sizing your cache](https://www.varnish-cache.org/docs/3.0/tutorial/sizing_your_cache.html)
- [Run-time parameters](https://www.varnish-cache.org/docs/trunk/reference/varnishd.html#run-time-parameters)
- [Grace mode](https://www.varnish-cache.org/trac/wiki/VCLExampleGrace)
- [vsm](https://www.varnish-cache.org/docs/trunk/reference/vsm.html)
- [varnish调优](http://onebitbug.me/2012/12/04/varnish-tune/)

