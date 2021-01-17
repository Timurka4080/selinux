# Инструкции
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

● Для начала установим нужные нам пакеты для раоты с __SELinux__ и сам __Nginx__

	yum update
	yum install epel-release && /
	  yum install nginx vim net-tools /
	  setools-console policycoreutils-python /
	  setroubleshoot-server

● Теперь зайдем в файл __nginx.conf__  и заменим там стандартный порт __80__ на любой другой запустим __Nginx__

	vim /etc/nginx/nginx.conf
	systemctl start nginx

● После того как увидем ощибку идем влоги и смотри их содержимое сразу укажу несколко способой сделать это 

	less /var/log/audit/audit.log
	audit2why < /var/log/audit/audit.log
	sealert -a /var/log/audit/audit.log

● Последний мне нрасится больше всего так как самый информативный. Далее опишу три способа решения данной проблемы 

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
