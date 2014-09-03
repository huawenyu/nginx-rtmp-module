
Documentation ref README-org.md

# QuickStart

# Building & Debug

## Pre-Assume

path:
  - nginx:       ~/workspace/nginx
  - rtmp-module: ~/workspace/nginx-rtmp-module
  - Install nginx into $HOME/nginx

## Build with Debug

```Shell
1. rebuild with debug info
$ cd ~/workspace/nginx
$ ./configure --prefix=$HOME/nginx --with-debug --with-cc-opt="-O0" \
	--add-module=$HOME/workspace/nginx-rtmp-module
$ make -j4
$ sudo make install

2. change worker_processes to 1               <<< simpler our debug, also hit breakpoint for every http-access
$ vi $HOME/nginx/conf/nginx.conf
worker_processes 1

3. restart nginx starts with one main process + 1 child process (as we specified in config).
$ sudo $HOME/nginx/sbin/nginx -s stop         <<< restart nginx
$ sudo $HOME/nginx/sbin/nginx -p $HOME/nginx  <<< use our compiled version
$ ps -ef | grep nginx                         <<< get <child_pid>, there are 2 nginx processes running. One Master, One Child. Get the pid of the child process.

4. using gdb attach our running process
$ gdb $HOME/nginx/sbin/gninx
(gdb) br ngx_http_<filename>_module.c:<line>
(gdb) attach <child_pid>
(gdb) continue
```

## Config & Running

Ref from [Stream Video](http://www.leaseweblabs.com/2013/11/streaming-video-demand-nginx-rtmp-module/)
  - Use flowplayer as our player
  - Use Chrome-browser as our remote-access
  - our www path: $HOME/www-root with two sub-dir flvs, mp4s which use to put the video files
  - and have [mp4s/bbb.mp4 video file](http://vod.leasewebcdn.com/bbb.mp4).

(1). modify nginx.conf
```Diff
Diff of $HOME/nginx/conf/nginx.conf
@@ -1,5 +1,5 @@

+# Grant the right to read our $HOME/www-root if we donnot use the default $HOME/nginx/html
-#user  nobody;
+user  root;
 worker_processes  1;

 #error_log  logs/error.log;
@@ -13,6 +13,24 @@
     worker_connections  1024;
 }

+# Grant port-1935 to rtmp
+rtmp {
+    server {
+        listen 1935;
+
+        chunk_size 4000;
+
+        # video on demand for flv files
+        application vod {
+            play /home/wilson/www-root/flvs;
+        }
+
+        # video on demand for mp4 files
+        application vod2 {
+            play /home/wilson/www-root/mp4s;
+        }
+    }
+}
+

 http {
     include       mime.types;
@@ -35,23 +53,21 @@
     server {
         listen       80;
         server_name  localhost;
+# if we donnot use the default $HOME/nginx/html, we should definitly give our www-root, also remove the main-page's path at below.
+        root         /home/wilson/www-root;

         #charset koi8-r;

         #access_log  logs/host.access.log  main;

         location / {
-            root   html;
+            #root   html;
             index  index.html index.htm;
         }

```

(2). Download flowplayer from Bottom of [flowplayer](http://flash.flowplayer.org/plugins/streaming/rtmp.html)
(3). cp sub-dir 'www-root' to $HOME, so we have $HOME/www-root
```
$ tree www-root
.
|-- flowplayer.rtmp-3.2.13.swf
|-- flvs
|-- index.html
|-- mp4s
|   `-- bbb.mp4
```
(4). copy index.html from [StandAlone](http://flash.flowplayer.org/demos/standalone/plugins/streaming/rtmp.html)
```javascript
<script>

$f("wowza", "http://releases.flowplayer.org/swf/flowplayer-3.2.18.swf", {

    clip: {
        // our video www-root/mp4s/bbb.mp4
        url: 'mp4:bbb.mp4',
        scaling: 'fit',
        // configure clip to use hddn as our provider, referring to our rtmp plugin
        provider: 'hddn'
    },

    // streaming plugins are configured under the plugins node
    plugins: {

        // here is our rtmp plugin configuration
        hddn: {
            url: "flowplayer.rtmp-3.2.13.swf",

            // netConnectionUrl defines where the streams are found
            // 172.16.80.127 is our rtmp-server nginx's ip address
            netConnectionUrl: 'rtmp://172.16.80.127:1935/vod2'
        }
    },
    canvas: {
        backgroundGradient: 'none'
    }
});
</script>

```
(5). restart nginx
```Shell
  $ sudo $HOME/nginx/sbin/nginx -s stop         <<< restart nginx
  $ sudo $HOME/nginx/sbin/nginx -p $HOME/nginx  <<< use our compiled version
```
