---
layout: post
title: "nginx配置缓存"
date: 2016-04-28
tags: Nginx
categories: Linux
---

### 配置记录笔记

```
http{
  .......
    proxy_connect_timeout 5;
    proxy_read_timeout 60;
    proxy_send_timeout 5;
    proxy_buffer_size 16k;
    proxy_buffers 4 64k;
    proxy_busy_buffers_size 128k;
    proxy_temp_file_write_size 128k;
    proxy_temp_path /tmp/proxy_temp_dir;
    proxy_cache_path /tmp/proxy_cache_dir levels=1:2 keys_zone=cache_one:200m inactive=1d             max_size=30g;
  ......
    
    server{
      ......
        location /static/ {    
          ......  
            proxy_cache cache_one;  
            proxy_cache_valid 200 302 1h;
            expires 10d;
          ......
             
        }
       ......     
    }
  ......
}
```