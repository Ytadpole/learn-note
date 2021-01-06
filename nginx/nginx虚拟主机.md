## nginx虚拟主机

#### 基于ip

添加ip别名

~~~shell
ifconfig eth0:2 172.27.0.43 broadcast 172.27.15.255 netmask 255.255.240.0 up
route add -host 172.27.0.43 dev eth0:1
~~~

nginx.conf配置server段

