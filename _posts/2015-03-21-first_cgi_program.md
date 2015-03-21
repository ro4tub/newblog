---
layout: post
title: 初尝CGI程序
---
#1. 安装nginx

	wget http://nginx.org/download/nginx-1.6.2.tar.gz
	tar zxvf nginx-1.6.2.tar.gz && cd nginx-1.6.2
	./configure && make && sudo make install
	sudo /usr/local/nginx/sbin/nginx

#2. 安装spawn_fastcgi


	git clone https://github.com/lighttpd/spawn-fcgi 
	cd spawn-fcgi
	./autogen.sh && ./configure && make
	sudo cp ./src/spawn-fcgi  /usr/local/nginx/sbin

#3. 安装fastcgi库

	wget  http://www.fastcgi.com/dist/fcgi.tar.gz
	tar zxvf fcgi.tar.gz && cd fcgi-2.4.1-SNAP-0311112127
	#修改include/fcgio.h增加#include <stdio.h>
	./configure && make && sudo make install

#4. 开发第一个CGI程序

	#include <fcgi_stdio.h>  
	#include <stdlib.h>  
 
	int main() {  
	    int count = 0;  
	    while (FCGI_Accept() >= 0) {  
	        printf("Content-type: text/html\r\n"  
	                "\r\n"  
	                ""  
	                "FastCGI Hello!"  
	                "Request number %d running on host%s "  
	                "Process ID: %d\n", ++count, getenv("SERVER_NAME"), getpid());  
	    }  
	    return 0;  
	}  

	gcc hello.c -o hello -lfcgi

#5. 发布和测试


	sudo mkdir /usr/local/nginx/cgi
	sudo cp hello /usr/local/nginx/cgi/
	oscar@ubuntu:~/work/hello_cgi$ /usr/local/nginx/sbin/spawn-fcgi  -a 127.0.0.1 -p 9527 -f /usr/local/nginx/cgi/hello
	spawn-fcgi: child spawned successfully: PID: 17157

	#修改nginx.conf
    location ~ \.cgi$ {
        fastcgi_pass 127.0.0.1:9527;
        fastcgi_index index.cgi;
        fastcgi_param SCRIPT_FILENAME fcgi$fastcgi_script_name;
        include fastcgi_params;
    }

	sudo kill -HUP PID


	oscar@ubuntu:~/work/hello_cgi$ curl http://localhost/hello.cgi
	FastCGI Hello!
	Request number 1 running on host localhost
	Process ID: 17341



#6. 参考资料


* http://stackoverflow.com/questions/4577453/fcgio-cpp50-error-eof-was-not-declared-in-this-scope