МОДУЛЬ 1

	1 Выполнить базовую настройку всех устройств
		а) Присвоить имена в соответствии с топологией
  			nano /etc/hostname
     			nano /etc/hosts
	
 	b, c, d) Рассчитайте ip адресацию IPv4, рассчитайте пул адрессов для branch, hq
        	ISP - HQR 10.1.1.0/30
        	ISP - BRR 10.2.2.0/30
        	HQR - HQSRV 192.168.100.0/26
        	BRR-BRSRV 192.168.200.0/28

        		nano /etc/network/interfaces
        		nano /etc/sysctl.conf (ISP, HQR, BRR)
        		sysctl -p
      
        		post-up iptables -A POSTROUTING -t nat -j MASQUERADE (ISP, HQR, BRR)
	  
   	2) Настройте внутреннюю динамическую маршрутизацию по средствам FRR
    			apt install frr
       			nano /etc/frr/daemons
			ospfd=yes
      			systemctl enable --now frr.service
        		vtysh
        			conft
        			ip router-id x.x.x.x
        			router ospf
        			network 10.1.1.0/30 area 10
        			network 192.168.100.0/26 area 10
        			do wr
										????????????????????????????????????????????
										vtysh
										conf t
										router ospf
										interface Tunnel1
										ip ospf network broadcast 
										end
										do wr
										exit
										systemctl restart frr
										????????????????????????????????????????????
	  	a) Составьте топологию сети L3
          		Сделать логическую схему сети
	    
	3) Настройке автоматическое распределение ip адрессов
        	а) Сделать резервацию для hqr-srv 
            		apt install isc-dhcp-server
            
            		nano /etc/default/isc-dhcp-server
              		interfacesv4="XX"

            		nano /etc/dhcp/dhcpd.conf
             			 subnet 192.168.100.0 netmask 255.255.255.192 {
               				 option routers 192.168.100.21;
               				 range 192.168.100.1 192.168.100.40;
              			}
           			 host HQ-SRV {
              				hardware ethernet
             				 fixed address
            			}

 	4) Настройте локальные учетные записи
        		su -l 
       			useradd Admin 
        		passwd Admin (и так с остальными) 
		        usermod -aG sudo Admin (если пользователя нужно добавить в sudo) 
		        nano /etc/passwd (для проверки создания пользователя)

     	5) Измерьте пропускную способность между двумя узлами hq-r isp по средствам утилиты iperf 3.
		          apt install iperf3
		          ----На ISP----
		            iperf3 -s
		          ----На HQ-R----
		            iperf3 -c 10.1.1.1

	6) Составьте backup скрипты для сохранения конфигурации сетевых устройств, а именно HQ-R BR-R.
		          mkdir /etc/back-up-save
		          nano /etc/back-up-save/script.sh
		            #!/bin/bash
		            DATE=$(date +%Y-%m-%d_%H-%M)
		            cp /etc/frr/frr.conf /etc/back-up-save/frr.conf-$DATE
         
	 			chmod +x script.sh
			        crontab -e
			        * * * * * /etc/back-up-save/script.sh

	7) Настройте подключение по SSH для удаленного конфигурирования устройства HQ-SRV по порту 2222. Учтите что вам необзодпмо перенаправить трафик.
		          apt install openssh-server
		          systemctl enable ssh
		          nano /etc/ssh/sshd_config
		          Port 2222
		          PermitRootLogin no
		          PasswordAuthentication yes
		          закрываем файл.
		          systemctl restart ssh
		          ssh student@192.168.1.2 -p 2222
		          post-up iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 2222
											////////////////////////////////////////////////
											sudo iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination <IP_HQ-SRV>:2222
											sudo iptables -A FORWARD -p tcp -d <IP_HQ-SRV> --dport 2222 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
											sudo iptables-save
											////////////////////////////////////////////////

	8) Настройте контроль доступа до HQ-SRV по SSH со всех кроме CLI
		          nano /etc/ssh/sshd_config
	    			sudo iptables -I OUTPUT -p tcp --dport 2222 -j DROP
		          AllowUsers student@192.168.0.1 student@192.168.0.140 student@192.168.0.129 (вводите ip и пользователя всех устройств кроме CLI)   



Модуль 2
      	
       1) Настройте DNS-сервер на сервере HQ-SRV:
            а. На DNS сервере необходимо настроить 2 зоны
               Зона hq.work, также не забудьте настроить обратную зону.
               ___________________________________________
               |  Имя            | Тип записи | Адрес    |
               | hq-r.hq.work    |   A, PTR   | IP-адрес |
               | hq-srv.hq.work  |   A, PTR   | IP-адрес |
               -------------------------------------------

              3oнa branch.work
               ______________________________________________
               |  Имя               | Тип записи | Адрес    |
               | br-r.branch.work   |   A, PTR   | IP-адрес |
               | br-srv.branch.work |   A        | IP-адрес |
               ----------------------------------------------
            nano /etc/bind/named.conf.options
                вместо 0.0.0.0 пишем 8.8.8.8
                listen-on-v6 { none; };
                listen-on { any; };

            nano /etc/resolv.conf
                  search hq.work 
                  search branch.work
                  nameserver 8.8.8.8

            nano  /etc/bind/named.conf.local
                zone "hq.work" {
	                    type master;
	                    file "/etc/bind/hq.db";
                };

                zone "branch.work" {
	                    type master;
	                    file "/etc/bind/branch.db";
                };

                zone "1.168.192.in-addr.arpa" {
	                    type master;
	                    file "/etc/bind/0.db";    - то что 1.168.192 это ваш ип только наоборот без последнего актета.
                };

                cp /etc/bind/db.127 /etc/bind/hq.db
            	cp /etc/bind/db.127 /etc/bind/branch.db
            	cp /etc/bind/db.127 /etc/bind/0.db

      		-------------
		перед этим заполнить резольв nameserver ip hqr-srv
  		chattr +i /etc/resolv.conf
  		-------------

![hq.db](https://i.imgur.com/kr0I2jk.png)
![brahch.db](https://i.imgur.com/Uik41EH.png)
![0.db](https://i.imgur.com/v8i88cP.png)

	2)  Настройте синхронизацию времени между сетевыми устройствами по протоколу NTP.
		а. В качестве сервера должен выступать роутер HQ-R
		   со стратумом 5
		Ь. Используйте Loopback интерфейс на HQ-R, как
		   источник сервера времени
		с. Все остальные устройства и сервера должны
		   синхронизировать свое время с роутером HQ-R
		d. Все устройства и сервера настроены на московский
		   часовой пояс (UTC +3)

		---HQ-R---
     		apt install chrony
       			nano /etc/chrony/chrony.conf
				server 127.0.0.1 iburst prefer
	   			hwtimestamp *
	      			local stratum 5
		 		allow all
    		--client-
      			nano /etc/chrony/chrony.conf
	 			server 10.1.1.2 iburst prefer
              
	5) Сконфигурируйте веб-сервер LMS Apache на сервере BR-SRV:
		a. На главной странице должен отражаться номер места
		b. Используйте базу данных mySQL
		c. Создайте пользователей в соответствии с таблицей, пароли у всех пользователей «P@ssw0rd»
		
  		apt install apache2 -y
    		apt install mariadb-server
	  		Заходим на оф.сайт moodle -> legacy realeases - > moodle 4.0.12 и качаем tgz
	    		Переходим в раздел куда скачался файл /root/Downloads
      		tar -xzvf moodle
		cp -r /moodle/ /var/www
  		cd /var/www/
    		rm -r /html/
      		cp -r /moodle/ /html/
    		chmod 777 /var/www/
      		rm -r moodle/
			apt install php -y
	  		apt install php-curl
	  		apt install php-zip
     			apt install php-mysqli
			apt install php-xml
     			apt install php-mbstring
			apt install php-gd
   			apt intstall php-intl
      			apt install php-xmlrpc
	 		apt install php-soap
			systemctl restart apache2
			
  		mysql -u root -p
    			Test123
       		CREATE USER 'moodle'@'localhost' IDENTIFIED BY 'password123';
	 	GRANT ALL PRIVILEGES ON *.* TO 'moodle'@'localhost' WITH GRANT OPTION;
		FLUSH PRIVILEGES;

  		Далее заходим на 127.0.0.1 и проходим все пункты. Если возникают ошибки то фиксим :)

      		Создать пользователей     -> 1. Администрирование -> 2. Пользователи -> 3. Добавить пользователя
		Создать глобальную группу -> 1. Администрирование -> 2. Пользователи -> 3. Глобальные группы -> 4. Добавить глобальную группу

    		
       			     
