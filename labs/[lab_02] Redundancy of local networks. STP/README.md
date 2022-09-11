# Избыточность локальных сетей. STP
## Физическая топология сети
![Физическая топология сети](https://github.com/Yabovbel/otus-networks/blob/80f1cb28cf172ef0bb89977af216cc536028fb79/labs/%5Blab_02%5D%20Redundancy%20of%20local%20networks.%20STP/pictures/otus-lab2-1.png "Физическая топология сети")
## Таблица адресации
Устройство	| Интерфейс	| IP-адрес	| Маска подсети
---		| ---		| ---		| ---
S1		| VLAN 1	| 192.168.1.1	| 255.255.255.0
S2		| VLAN 1	| 192.168.1.2	| 255.255.255.0
S3		| VLAN 1	| 192.168.1.3	| 255.255.255.0

## Ход выполнения лабораторной работы
1. __Настройка базовых параметров коммутатора__

	Настройка на примере __S1__. Аналогичные настройки выполняем на коммутаторах __S2__ и __S3__  
	* Отклчаем domain-lookup чтобы IOS не интерпретировала неправильно введенные команды как имя хоста и не пыталась их разрешить, обращаясь к DNS серверу.
	`Switch(config)#no ip domain-lookup`  
	* Присваиваем имя устройства командой `Switch(config)#hostname S1`  
	* Ограничение доступа к привелегированному режиму шифрованным паролем *class* командой `S1(config)#enable secret class`  
	* Назначение пароля *cisco* для доступа через консоль и виртуальные интерфейсы  
	```
	S1(config)#line vty 0 15
	S1(config-line)#password cisco
	S1(config-line)#login
	S1(config-line)#exit
	S1(config)#line console 0
	S1(config-line)#password cisco
	S1(config-line)#login
	S1(config-line)#exit
	```
	* Включаем шифрование паролей в конфигурации командой `S1(config)#service password-encryption`  
	* Для консольного интерфейса просим IOS восстанавливать набранную команду, если ввод был прерван системным сообoением из syslog  
	```
	S1(config)#line con 0
	S1(config-line)#logging synchronous 
	S1(config-line)#exit
	```
	* Настраиваем приветственное сообщение о запрете неавторизованного доступа `S1(config)#banner motd #unauthorized access is prohibited#`  
	* По таблице адресации задаем IP адрес SVI интерфесу для VLAN 1 чтобы можно было управлять коммутатором, поднимаем SVI.  
	```
	S1(config)#int vl 1
	S1(config-if)#ip a 192.168.1.1 255.255.255.0
	S1(config-if)#no sh
	```
	* Проверяем корректность настроенной адресации
		* Для __S1__  
		```
		S1#sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
							.....
		Vlan1                  192.168.1.1     YES manual up                    up
		```
		* Для __S2__
		```
		S2#sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
							.....
		Vlan1                  192.168.1.2     YES manual up                    up
		```
		* Для __S3__
		```
		S3#sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
							.....
		Vlan1                  192.168.1.3     YES manual up                    up
		```
	* Дополнительно настроем вручную время, чтобы было реально найти что-то по логам при необходимости  
	```
	S1#clock set 22:24:00 09 september 2022
	S1#show clock 
	22:24:7.914 UTC Fri Sep 9 2022
	```
	* Сохраняем текущую конфигурацию в NVRAM
	```
	S1#cop run sta
	Destination filename [startup-config]? 
	Building configuration...
	[OK]
	```  
2. __Проверка работоспособности сети и прохождения эхо-запросов между коммутаторами__

	* ping от коммутатора __S1__ на коммутатор __S2__
	```
	S1#ping 192.168.1.2
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms
	```
	* ping от коммутатора __S1__ на коммутатор __S3__
	```
	S1#ping 192.168.1.2
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms
	```
	* ping от коммутатора __S2__ на коммутатор __S3__
	```
	S2#ping 192.168.1.3
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.3, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms
	```
3. __Определение корневого моста__

	Настройка на примере __S1__. Настройки для __S2__ и __S3__ аналогичны.
	* Отключение всех портов на комутаторах  
	
		По материалам предыдущей лабораторной работы применим ряд настроек для обеспечения бОльшей безопасности
		* Создаем VLAN, куда заведем все неиспользуемые порты
		```
		S1(config)#vl 8
		S1(config-vlan)#name ParkingLot
		S1(config-vlan)#exit
		```
		* Все порты на коммутаторе переводим в созданный VLAN и включаем DTP.
		```
		S1(config)#int range fa0/1-24,gi0/1-2
		S1(config-if-range)#switchport mode access 
		S1(config-if-range)#switchport access vlan 8
		S1(config-if-range)#switchport nonegotiate 
		```
		* Непосредственно выключаем порты.
		```
		S1(config-if-range)#shutdown 
		S1(config-if-range)#exit
		```
		* Переводим VTP в прозрачный режим
		```
		S1(config)#vtp mode transparent 
		Setting device to VTP TRANSPARENT mode.
		```
	* Настройка портов, участвующих в топологии, в качестве транковых
		* Интерфейсы с __F0/1__ по __F0/4__ переводим в режим транка
		```
		S1(config)#int range fa0/1-4
		S1(config-if-range)#switchport mode trunk 
		```
		* Назначаем VLAN 1 в качестве native и разрешаем его прохождение
		```
		S1(config-if-range)#switchport trunk native vlan 1
		S1(config-if-range)#switchport trunk allowed vlan 1
		```
		* Снимаем access VLAN
		```
		S1(config-if-range)#no switchport access vlan
		S1(config-if-range)#exit
		```
	* Включаем порты __F0/2__ и __F0/4__ на всех коммутаторах
		```
		S1(config)#int range fa0/2,fa0/4
		S1(config-if-range)#no sh
		```
	* Просмотр данных протокола STP
		* Для __S1__
		```
		S1#sh spanning-tree 
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 This bridge is the root
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0001.4364.4AC1
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/4            Desg FWD 19        128.4    P2p
		Fa0/2            Desg FWD 19        128.2    P2p
		```
		* Для __S2__
		```
		S2#sh spanning-tree 
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 Cost        19
					 Port        2(FastEthernet0/2)
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0003.E40E.9B2B
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/2            Root FWD 19        128.2    P2p
		Fa0/4            Desg FWD 19        128.4    P2p
		```
		* Для __S3__
		```
		S3#sh spanning-tree 
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 Cost        19
					 Port        4(FastEthernet0/4)
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0060.3EAE.A365
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/4            Root FWD 19        128.4    P2p
		Fa0/2            Altn BLK 19        128.2    P2p
		```
	### Схема выбора портов протоколом STP, исходя из MAC-адресов коммутаторов
	![Схема выбора портов протоколом STP, исходя из MAC-адресов коммутаторов](https://github.com/Yabovbel/otus-networks/blob/80f1cb28cf172ef0bb89977af216cc536028fb79/labs/%5Blab_02%5D%20Redundancy%20of%20local%20networks.%20STP/pictures/otus-lab2-2.png "Схема выбора портов протоколом STP, исходя из MAC-адресов коммутаторов")
	
	---
	Корневым коммутатором является __S1__.  

	Коммуатор __S1__ был выбран протоколом STP в качестве корневого, так как Priority на всех коммутаторах оставлены по умолчанию (32769), но при этом у __S1__ наименьший MAC-адрес.  

	Корневыми портами являются: порт __F0/2__ на коммутаторе __S2__ и порт __F0/4__ на коммутаторе __S3__.  

	Назначенными портами являются все порты на корневом коммутаторе, то есть __F0/2__ и __F0/4__ на коммутаторе __S1__, также назначенным портом является __F0/4__ на коомутаторе __S2__.

	В качестве альтернатвноего порта, который в данный момент заблокирован, протоколом STP был выбран порт __F0/2__ на коммутаторе __S3__.  
	STA выбрал данный порт в качестве заблокированного, по следующим причинам:  
	1. Данный порт принадлежит сегменту между коммутаторами __S2__ и __S3__, где ни один из коммутаторов не является корневым.  
	2. Стоимость до корневого коммутатора от коммутаторов __S2__ и __S3__ одинаковая и равна 19. Соответсвенно назначенный/альтернативный порт не могут быть выбраны на основе наименьшей стоимости от коммутатора до корневого.  
	3. Выбор произведен на основе MAC-адресов коммутаторов __S2__ и __S3__. Порт, принадлежащий коммуататору __S2__, у которого меньший MAC-адрес стал назначенным, а порт на коммутаторе __S3__ перешел в статус альтернативного.  
	---
4. __Наблюдение за процессом выбора протоколом STP порта, исходя из стоимости портов__

	* Фиксируем вывод команды show spanning-tree до внесения изменений в стоимость поротов с некорневых коммутаторов __S2__ и __S3__.
		* Для __S2__
		```
		S2#sh spanning-tree 
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 Cost        19
					 Port        2(FastEthernet0/2)
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0003.E40E.9B2B
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/2            Root FWD 19        128.2    P2p
		Fa0/4            Desg FWD 19        128.4    P2p
		```
		* Для __S3__
		```
		S3#sh spanning-tree 
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 Cost        19
					 Port        4(FastEthernet0/4)
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0060.3EAE.A365
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/4            Root FWD 19        128.4    P2p
		Fa0/2            Altn BLK 19        128.2    P2p
		```
	* Изменяем стоимость корневого порта __F0/4__ на коммутаторе с самым высомим BID, то есть __S3__ (наибольший MAC-адрес)  
	```
	S3(config)#int fa0/4
	S3(config-if)#spanning-tree vlan 1 cost 18
	```
	__Внимание!__  
	__Данная в инструкции к лабораторной работе команда `spanning-tree cost 18` не работает в Cisco Packet Tracer 8.0.__  
	__Для корректной работы следует указывать VLAN для которого задаем cost.__  
	__Пример команды: `spanning-tree vlan 1 cost 18`__
	
	* Вывод команды show spanning-tree c некорневых коммутаторов __S2__ и __S3__.
		* Для __S2__
		```
		S2#sh spanning-tree 
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 Cost        19
					 Port        2(FastEthernet0/2)
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0003.E40E.9B2B
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/2            Root FWD 19        128.2    P2p
		Fa0/4            Altn FWD 19        128.4    P2p
		```
		* Для __S3__
		```
		S3#sh spanning-tree 
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 Cost        18
					 Port        4(FastEthernet0/4)
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0060.3EAE.A365
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/4            Root FWD 18        128.4    P2p
		Fa0/2            Desg BLK 19        128.2    P2p
		```
	### Схема выбора портов протоколом STP, исходя из стоимости портов
	![Схема выбора портов протоколом STP, исходя из стоимости портов](https://github.com/Yabovbel/otus-networks/blob/80f1cb28cf172ef0bb89977af216cc536028fb79/labs/%5Blab_02%5D%20Redundancy%20of%20local%20networks.%20STP/pictures/otus-lab2-3.png "Схема выбора портов протоколом STP, исходя из стоимости портов")
	
	---
	На сегменте между коммутаторами __S2__ и __S3__ назначенный и заблокированный порты поменялись местами.  
	
	Так как на сегменте между коммутаторами __S2__ и __S3__ ни один из коммутаторов не является корневым. То выбор коммутатора, на котором будет назначенный порт зависит от стоимости от этих коммутаторов до корневого.  
	Порт __F0/2__ коммутатора __S3__ с наименьшей стоимостью пути к корневому коммутатору (18) является назначенным портом.
	
	---
	* Удаление стоимости порта на интерфейсе F0/4 коммутатора S3
	```
	S3(config)#int fa0/4
	S3(config-if)#no spanning-tree vlan 1 cost
	```
5. __Наблюдение за процессом выбора протоколом STP порта, исходя из приоритета портов__

	* Включаем порты __F0/1__ и __F0/3__ на всех коммутаторах
	```
	S1(config)#int range fa0/1,fa0/3
	S1(config-if-range)#no sh
	```
	* Просмотр данных протокола STP
		* Для __S1__
		```
		S1#sh sp
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 This bridge is the root
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0001.4364.4AC1
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/1            Desg FWD 19        128.1    P2p
		Fa0/2            Desg FWD 19        128.2    P2p
		Fa0/3            Desg FWD 19        128.3    P2p
		Fa0/4            Desg FWD 19        128.4    P2p
		```
		* Для __S2__
		```
		S2#sh sp
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 Cost        19
					 Port        1(FastEthernet0/1)
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0003.E40E.9B2B
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/2            Altn BLK 19        128.2    P2p
		Fa0/1            Root FWD 19        128.1    P2p
		Fa0/3            Desg FWD 19        128.3    P2p
		Fa0/4            Desg FWD 19        128.4    P2p
		```
		* Для __S3__
		```
		S3#sh sp
		VLAN0001
		  Spanning tree enabled protocol ieee
		  Root ID    Priority    32769
					 Address     0001.4364.4AC1
					 Cost        19
					 Port        3(FastEthernet0/3)
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

		  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
					 Address     0060.3EAE.A365
					 Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
					 Aging Time  20

		Interface        Role Sts Cost      Prio.Nbr Type
		---------------- ---- --- --------- -------- --------------------------------
		Fa0/3            Root FWD 19        128.3    P2p
		Fa0/2            Altn BLK 19        128.2    P2p
		Fa0/1            Altn BLK 19        128.1    P2p
		Fa0/4            Altn BLK 19        128.4    P2p
		```
	### Схема выбора портов протоколом STP, исходя из приоритета портов
	![Схема выбора портов протоколом STP, исходя из приоритета портов](https://github.com/Yabovbel/otus-networks/blob/80f1cb28cf172ef0bb89977af216cc536028fb79/labs/%5Blab_02%5D%20Redundancy%20of%20local%20networks.%20STP/pictures/otus-lab2-4.png "Схема выбора портов протоколом STP, исходя из приоритета портов")
	
	---
	На некорневых коммутаторах в качестве корневого порта протоколом STP были выбраны следующие порты:  
	На коммутаторе __S2__ порт __F0/1__; На коммуаторе __S3__ порт __F0/3__.  
	
	Данные порты были выбраны протоколом STP в качестве корневых, так как порты отноятся к сегментам между корневым и некорневым коммутатороом (сегмент между __S1__ и __S2__; сегмент между __S1__ и __S3__).  
	Если на таком сегменте присутсвует несколько линков, то приоритет отдается линку с наименьшим номером порта на корневом коммутаторе.  
	Соответвенно, так как от __S1__ в сторону __S2__ два линка, подключенных в порты __F0/1__ и __F0/2__ на __S1__, то коммутатор __S2__ назначит корневым порт, соединенный с портом __F0/1__ на __S1__.  
	Отсюда на __S2__ порт __F0/1__ - корневой, __F0/2__ - заблокированный.  
	Аналогиная ситуация между коммутаторами __S1__ и __S3__. Так как от __S1__ в сторону __S3__ два линка, подключенных в порты __F0/3__ и __F0/4__ на __S1__, то коммутатор __S3__ назначит корневым порт, соединенный с портом __F0/3__ на __S1__.  
	Отсюда на __S3__ порт __F0/3__ - корневой, __F0/4__ - заблокированный.  
	
	---
6. __Обобщение порядка выбора портов протоколом STP__

	* Какое значение протокол STP использует первым после выбора корневого моста, чтобы определить выбор порта? -- __Cost (выбирается наименьшее)__  
	* Если первое значение на двух портах одинаково, какое следующее значение будет использовать протокол STP при выборе порта? -- __Priority (выбирается наименьшее)__  
	* Если оба значения на двух портах равны, каким будет следующее значение, которое использует протокол STP при выборе порта? -- __Номер порта на корневом коммутаторе (выбирается наименьшее)__  
