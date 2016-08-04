# ansible：playbook定制Nginx安装配置
------------------------------------------------
playbook是通过YAML格式来进行描述定义的，可以实现多台主机应用的部署，定义在olly_servers组上执行特定指令步骤。下面介绍一个基本的playbook示例：


```
	
    ---
	- hosts： olly_servers
	  vars：
	    worker_processes： 4
	    num_cpus： 4
	    max_open_file： 65506
	    root： /data
	  remote_user： root
	  tasks：
	  - name： ensure nginx is at the latest version
	    yum： pkg=nginx state=latest
	  - name： write the nginx config file
	    template： src=/home/test/ansible/nginx/nginx2.conf dest=/etc/nginx/nginx.conf
	    notify：
	    - restart nginx
	  - name： ensure nginx is running
	    service： name=nginx state=started
	  handlers：
	    - name： restart nginx
	      service： name=nginx state=restarted 

```

以上playbook定制了一个简单的Nginx软件包管理，内容包括安装、配置模板、状态管理等。下面详细对上面这段配置进行说明。

###1 定义主机与用户

在playbook执行时，可以为主机或组定义变量，比如指定远程登录用户。以下为olly_servers组定义的相关变量，变量的作用域只限于olly_servers组下的主机。同时通过vars参数定义了4个变量（配置模板用到）

```

	- hosts： olly_servers
	  vars：
	    worker_processes： 4
	    num_cpus： 4
	    max_open_file： 65506
	    root： /data
        server_hostname: www.example.com
	  remote_user： root 

```
###2 任务列表

要执行的任务在`tasks：`下定义，且每个任务都有一个任务名 `name：`playbook将按定义的配置文件自上而下的顺序执行每个任务。

在以上例子中 ansible需要执行的任务有3个，分别如下：

任务1：----安装nginx

	  - name： ensure nginx is at the latest version
	    yum： pkg=nginx state=latest

任务2：----把主控机上的nginx.conf模版配置文件 进行“渲染”，并拷贝到远程机器上

	  - name： write the nginx config file
	    template： src=/home/test/ansible/nginx/nginx2.conf dest=/etc/nginx/nginx.conf
	    notify：
	    - restart nginx

/home/test/ansible/nginx/nginx2.conf文件中需要渲染的内容如下

```

		user nobody nogroup;
		
		
		user              nginx;
		worker_processes  {{ worker_processes }};
		{% if num_cpus == 2 %}
		worker_cpu_affinity 01 10;
		{% elif num_cpus == 4 %}
		worker_cpu_affinity 1000 0100 0010 0001;
		{% elif num_cpus >= 8 %}
		worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
		{% else %}
		worker_cpu_affinity 1000 0100 0010 0001;
		{% endif %}
		worker_rlimit_nofile {{ max_open_file }};
		
		error_log /data/error.log crit;
		pid /var/nginx.pid;
		
		
		events {
			use epoll;
			worker_connections 10240;
		}
		
		http {
			server_tokens off;
			server_tag off;
			autoindex off;
			access_log off;
			include mime.types;
			default_type application/octet-stream;
		
			server_names_hash_bucket_size 128;
			client_header_buffer_size 32k;
			large_client_header_buffers 4 32k;
			client_max_body_size 10m;
			client_body_buffer_size 256k;
		
			sendfile on;
			tcp_nopush on;
			keepalive_timeout 60;
			tcp_nodelay on;
		
			gzip on;
			gzip_min_length 1k;
			gzip_buffers 4 16k;
			gzip_http_version 1.0;
			gzip_comp_level 2;
			gzip_types text/plain application/x-javascript text/css application/xml;
			gzip_vary on;
		
			proxy_connect_timeout 600;
			proxy_read_timeout 600;
			proxy_send_timeout 600;
			proxy_buffer_size 128k;
			proxy_buffers 4 128k;
			proxy_busy_buffers_size 256k;
			proxy_temp_file_write_size 256k;
			proxy_headers_hash_max_size 1024;
			proxy_headers_hash_bucket_size 128;
		
			#proxy_redirect off;
			proxy_set_header Host $host;
			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header REMOTE-HOST $remote_addr;
			proxy_set_header X-Forwarded-For $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		
			proxy_temp_path /data/nginx_temp/nginx_temp;
			proxy_cache_path /data/nginx_temp/nginx_cache levels=1:2 keys_zone=cache_one:2048m inactive=30m max_size=60g;
		
		
		
		    server {
		            listen       80 default_server;
		            server_name  {{ server_hostname }};
		            root /srv/wordpress/ ;
		     
		        client_max_body_size 64M;
		     
		        # Deny access to any files with a .php extension in the uploads directory
		            location ~* /(?:uploads|files)/.*\.php$ {
		                    deny all;
		            }
		     
		            location / {
		                    index index.php index.html index.htm;
		                    try_files $uri $uri/ /index.php?$args;
		            }
		     
		            location ~* \.(gif|jpg|jpeg|png|css|js)$ {
		                    expires max;
		            }
		     
		            location ~ \.php$ {
		                    try_files $uri =404;
		                    fastcgi_split_path_info ^(.+\.php)(/.+)$;
		                    fastcgi_index index.php;
		                    fastcgi_pass  unix:/var/run/php-fpm/wordpress.sock;
		                    fastcgi_param   SCRIPT_FILENAME
		                                    $document_root$fastcgi_script_name;
		                    include       fastcgi_params;
		            }
		    }
		
		        
			log_format access '$remote_addr - $remote_user [$time_local] "$request"'
				'$status $body_bytes_sent "$http_referer"'
				'"$http_user_agent" $http_x_forwarded_for';
		
		
		}


```

Ansible会根据定义好的模板渲染成真实的配置文件，模板使用YAML语法，最终生成的nginx.conf配置如下：

```

	user              nginx；
	worker_processes  4；
	worker_cpu_affinity 1000 0100 0010 0001；
	worker_rlimit_nofile 65506；
	… … 

```

当目标主机配置文件发生变化后，通知处理程序（Handlers）来触发后续的动作，比如重启nginx服务。Handlers中定义的处理程序在没有通知触发时是不会执行的，触发后也只会运行一次。触发是通过Handlers定义的name标签来识别的，比如下面notify中的“restart nginx”与handlers中的“name：restart nginx”保持一致。

```

	notify：
	    - restart nginx
	handlers：
	    - name： restart nginx
	      service： name=nginx state=restarted 

```

### 执行playbook

命令如下：

```

	ansible-playbook /home/test/ansible/playbooks/nginx.yml -f 3

```

`-f 3`代表启动3个进程并发执行。