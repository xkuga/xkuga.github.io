---
layout: post
title: Wordpress XHProf XHGui 性能分析
date:  2016-08-07
author: kuga
---

最近发现用 Wordpress 搭建的博客响应很慢，特别是后台，TTFB(Time To First Byte) 时间很长。
这个 TTFB 其实就是 PHP 代码的执行时间，后来结合使用 XHProf 和 XHGui 分析之后，
发现这些都是由 WP_Http_Streams::request 这个函数造成的，如下图。

![xhgui](/img/xhgui.jpg)

加了日志发现，这些都是 Wordpress 和插件相关的更新请求。

```
Array
(
    [uri] => /wp-cron.php?doing_wp_cron=1470375031.1464920043945312500000
    [req] => https://api.wordpress.org/plugins/update-check/1.1/
    [cost] => 1.3579370975494
    [date] => 2016-08-05 05:30:33
)
Array
(
    [uri] => /wp-admin/
    [req] => http://www.getbeans.io/rest-api/
    [cost] => 5.5579149723053
    [date] => 2016-08-04 16:10:12
)
Array
(
    [uri] => /wp-admin/admin-ajax.php?action=dashboard-widgets&widget=dashboard_primary&pagenow=dashboard
    [req] => http://wordpress.org/plugins/rss/browse/popular/
    [cost] => 1.7584221363068
    [date] => 2016-08-04 16:10:20
)
```

在 wp-config.php 添加下面的代码后(关闭更新)，TTFB 平均为 0.2~0.3 秒。

```php
define('AUTOMATIC_UPDATER_DISABLED', true);
add_filter('automatic_updater_disabled', '__return_true');
```

## Docker
---------

这次也用 Docker 搭建了 Ｗordpress 和 XHGui，源码在<a href="https://github.com/xkuga/docker-wordpress-xhgui" target="_blank">这里</a>，下面主要说一下碰到的问题。

#### 环境变量问题

这次是通过 docker-compose 去连接 mysql 和 mongodb 的，所以容器中会自动包含相应的环境变量。
本来是打算在 wp-config.php 中使用 getenv 来获取环境变量的，但失败了。
原因是 php-fpm 默认是清空环境变量的，他的配置在下面这个文件中。

```
/etc/php5/fpm/pool.d/www.conf
```

找到 clear_env 这个选项。

```
; Clear environment in FPM workers
; Prevents arbitrary environment variables from reaching FPM worker processes
; by clearing the environment in workers before env vars specified in this
; pool configuration are added.
; Setting to "no" will make all environment variables available to PHP code
; via getenv(), $_ENV and $_SERVER.
; Default Value: yes
;clear_env = no
```

如果你找不到上面这个选项，那是因为 PHP 版本不对，下面是官方文档。

```
clear_env boolean
Clear environment in FPM workers.
Prevents arbitrary environment variables from reaching FPM worker processes
by clearing the environment in workers before env vars specified in this pool configuration are added.
Since PHP 5.4.27, 5.5.11, and 5.6.0. Default value: Yes.
```

可见，默认 clear_env 是 yes，这里要把他设置为 no 才能在 php-fpm 中获取环境变量。
但由于这个可能会有安全问题，所以我改为在 entrypoint.sh 脚本处理。
虽然可能不是 Best Practice，但至少 It works!

关于这个问题，GitHub 中也有相关的 <a href="https://github.com/docker-library/php/issues/74" target="_blank">issue</a>。

#### docker-compose no such image

docker-compose up 的时候，明明镜像是存在的，但他还是提示 no such image。
这是因为，他引用到了那些之前被删除的镜像，运行下面这个命令可以解决。

```
docker-compose rm
```

关于这个问题，GitHub 中也有相关的 <a href="https://github.com/docker/compose/issues/1113" target="_blank">issue</a>。

## 引用
------

如果想要一步一步了解如何部署 Wordpress 和 XHGui，可以参考下面的文章。

1. <a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-xhprof-and-xhgui-for-profiling-php-applications-on-ubuntu-14-04" target="_blank">https://www.digitalocean.com/community/tutorials/how-to-set-up-xhprof-and-xhgui-for-profiling-php-applications-on-ubuntu-14-04</a>
2. <a href="https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-on-ubuntu-14-04" target="_blank">https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-on-ubuntu-14-04</a>
