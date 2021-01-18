## Инструкции
● Для проверки надо скачать репозиторий 

	cd ~
    git clone git@github.com:Timurka4080/selinux.git
	
● После этого зайдите в директорию __selinux__ и запускаем виртуальную машину с помощью vagrant

	cd ~/selinux/
	vagrant up
## Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

● Для начала установим нужные нам пакеты для работы с __SELinux__ и сам __Nginx__

	yum update
	yum install epel-release && /
	  yum install nginx vim net-tools /
	  setools-console policycoreutils-python /
	  setroubleshoot-server

● Теперь зайдем в файл __nginx.conf__  и заменим там стандартный порт __80__ на любой другой запустим __Nginx__

	vim /etc/nginx/nginx.conf
	systemctl start nginx

● После того как увидим ошибку, идем в логи и смотри их содержимое. Сразу укажу несколько способов как сделать это  

	less /var/log/audit/audit.log
	audit2why < /var/log/audit/audit.log
	sealert -a /var/log/audit/audit.log

● Последний мне нравится больше всего так как самый информативный. Далее опишу три способа решения данной проблемы 

Если вы хотите разрешить /usr/sbin/nginx для привязки к сетевому порту $PORT_ЧИСЛО
То вы должны добавить номер порта в контекст. Где PORT_TYPE может принимать значения: 
http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t. Конкретно в нашем случае __http_port_t__

Сделать

	semanage port -a -t PORT_TYPE -p tcp "port"


Если хотите allow nis to enabled
То вы должны сообщить SELinux об этом, включив переключатель «nis_enabled».

Сделать

	setsebool -P nis_enabled 1

Если вы считаете, что nginx должно быть разрешено name_bind доступ к port 3502 tcp_socket по умолчанию.
То рекомендуется создать отчет об ошибке.
Чтобы разрешить доступ, можно создать локальный модуль политики.

Сделать


	ausearch -c 'nginx' --raw | audit2allow -M my-nginx
	semodule -i my-nginx.pp

## Обеспечить работоспособность приложения при включенном selinux.
● Запускаем тестовый сенд и делаем набор команд из инструкции на клиентской стороне 

	[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
	> server 192.168.50.10
	> zone ddns.lab
	> update add www.ddns.lab. 60 A 192.168.50.15
	> send
	update failed: SERVFAIL
	> quit

● Потом идем на вторую виртуалку с серверной частью и запускаем там следующую команду

	audit2why < /var/log/audit/audit.log

● Получаем вывод 

	[root@ns01 vagrant]# audit2why < /var/log/audit/audit.log
	type=AVC msg=audit(1610975676.442:1911): avc:  denied  { create } for  pid=5011 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.		

● И начинаем очень долго страдать. ОЧЕНЬ.... ДОЛГО.... СТРАДАТЬ....  
и когда надежда уже почти покинула случайно найти [ссылку](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-bind-configuration_examples/) на эту чудесную статью   
понять что у нас __named.ddns.lab.view1.jnl__ который находится в директории `/etc/named/dynamic/named.ddns.lab.view1` который мы нашли благодаря файлу `/etc/named.conf` а должен находиться в `/var/named/dynamic/` c контекстом `named_zone_t ` а у нас 

	[root@ns01 vagrant]# ll -Z /etc/named/dynamic/
	-rw-rw----. named named system_u:object_r:etc_t:s0 named.ddns.lab
	-rw-r--r--. named named system_u:object_r:etc_t:s0 named.ddns.lab.view1
	-rw-r--r--. named named system_u:object_r:etc_t:s0 named.ddns.lab.view1.jnl

● Делаем следующие команды 

	[root@ns01 vagrant]# semanage fcontext -a -t named_cache_t '/etc/named/dynamic(/.*)?'	
	[root@ns01 vagrant]# restorecon -R -v /etc/named/dynamic/

И все работает УРА.	

