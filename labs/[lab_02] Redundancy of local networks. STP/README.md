# Избыточность локальных сетей. STP
## Физическая топология сети

## Таблица адресации

1. Настройка базовых параметров коммутатора 
	Настройка на примере S1. Аналогичные настройки выполняем на коммутаторах S2 и S3
	1. Отклчаем domain-lookup чтобы IOS не интерпретировала неправильно введенные команды как имя хоста и не пыталась их разрешить, обращаясь к DNS серверу.
	Switch(config)#no ip domain-lookup 
	2. Пристваиваем имя устройства командой Switch(config)#hostname S1
	3. Ограничение доступа к привелегированному режиму шифрованным паролем *class* командой S1(config)#enable secret class
	4. Назначение пароля *cisco* для доступа через консоль и виртуальные интерфейсы
	S1(config)#line vty 0 15
	S1(config-line)#password cisco
	S1(config-line)#login
	S1(config-line)#exit
	S1(config)#line console 0
	S1(config-line)#password cisco
	S1(config-line)#login
	S1(config-line)#exit
	Включаем шифрование паролей в конфигурации командой
	S1(config)#service password-encryption
	5. Для консольного интерфейса просим IOS восстанавливать набранную команду, если ввод был прерван системным сообoением из syslog
	S1(config)#line con 0
	S1(config-line)#logging synchronous 
	S1(config-line)#exit
	6. Настраиваем приветсятвенное сообщение о запрете неавторизованного доступа S1(config)#banner motd #unauthorized access is prohibited#
	7. По таблице адресации задаем IP адрес SVI интерфесу для VLAN 1 чтобы можно было управлять коммутатором, поднимаем SVI.
	S1(config)#int vl 1
	S1(config-if)#ip a 192.168.1.1 255.255.255.0
	S1(config-if)#no sh
	Проверяем корректность настроенной адресации
		Для S1
		S1#sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
		.....
		Vlan1                  192.168.1.1     YES manual up                    up
		Для S2
		S2#sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
		.....
		Vlan1                  192.168.1.2     YES manual up                    up
		Для S3
		S3#sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
											.....
		Vlan1                  192.168.1.3     YES manual up                    up
	8. Дополнительно настроем вручную время, чтобы было реально найти что-то по логам при необходимости
	S1#clock set 22:24:00 09 september 2022
	S1#show clock 
	22:24:7.914 UTC Fri Sep 9 2022
	9. Сохраняем текущую конфигурацию в NVRAM
	S1#cop run sta
	Destination filename [startup-config]? 
	Building configuration...
	[OK]
2. Проверка работоспособности сети и прохождения эхо-запросов между коммутаторами
	ping от коммутатора S1 на коммутатор S2
	S1#ping 192.168.1.2
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms
	ping от коммутатора S1 на коммутатор S3
	S1#ping 192.168.1.2
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.2, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms
	ping от коммутатора S2 на коммутатор S3
	S2#ping 192.168.1.3
	Type escape sequence to abort.
	Sending 5, 100-byte ICMP Echos to 192.168.1.3, timeout is 2 seconds:
	!!!!!
	Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms
3.Определение корневого моста
	Настройка на примера S1. Настройки для S2 и S3 аналогичны.
	1. Отключение всех портов на комутаторах
		По материалам предыдущей лабораторной работы применим ряд настроек для обеспечения бОльшей безопасности
		Создаем VLAN, куда заведем все неиспользуемые порты
		S1(config)#vl 8
		S1(config-vlan)#name ParkingLot
		S1(config-vlan)#exit
		Все порты на коммутаторе переводим в созданный VLAN и включаем DTP.
		S1(config)#int range fa0/1-24,gi0/1-2
		S1(config-if-range)#switchport mode access 
		S1(config-if-range)#switchport access vlan 8
		S1(config-if-range)#switchport nonegotiate 
		Непосредственно выключаем порты.
		S1(config-if-range)#shutdown 
		S1(config-if-range)#exit
		Переводим VTP в прозрачный режим
		S1(config)#vtp mode transparent 
		Setting device to VTP TRANSPARENT mode.
	2. Настройка портов, участвующих в топологии, в качестве транковых
		S1(config)#int range fa0/1-4
		S1(config-if-range)#switchport mode trunk 
		Назначаем VLAN 1 в качестве native и разрешаем его прохождение
		S1(config-if-range)#switchport trunk native vlan 1
		S1(config-if-range)#switchport trunk allowed vlan 1
		Снимаем access VLAN
		S1(config-if-range)#no switchport access vlan
		S1(config-if-range)#exit 
	3. Включаем порты F0/2 и F0/4 на всех коммутаторах
		S1(config)#int range fa0/2,fa0/4
		S1(config-if-range)#no sh
	4. Просмотр данных протокола STP
		Для S1
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
		Для S2
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
		Для S3
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

		Корневым коммутатором является S1. 
		Коммуатор S1 был выбран протоколом STP в качестве корневого, так как Priority на всех коммутаторах оставлены по умолчанию (32769), но при этом у S1 наименьший MAC-адрес.
		Корневыми портами являются: порт F0/2 на коммутаторе S2 и порт F0/4 на коммутаторе S3
		Назначенными портами являются все порты на корневом коммутаторе, то есть F0/2 и F0/4 на коммутаторе S1, также назначенным портом является F0/4 на коомутаторе S2
		В качестве альтернатвноего порта, который в данный момент заблокирован, протоколом STP был выбран порт F0/2 на коммутаторе S3.
		STA выбрал данный порт в качестве заблокированного, по следующим причинам:
		1. Данный порт принадлежит сегменту между коммутаторами S2 и S3, где ни один из коммутаторов не является корневым
		2. Стоимость до корневого коммутатора от коммутаторов S2 и S3 одинаковая и равна 19. Соответсвенно назначенный/альтернативный порт не могут быть выбраны на основе наименьшей стоимости от коммутатора до корневого.
		3. Выбор произведен на основе MAC-адресов коммутаторов S2 и S3. Порт, принадлежащий коммуататору S2, у которого меньший MAC-адрес стал назначенным, а порт на коммутаторе S3 перешел в статус альтернативного.
4. Наблюдение за процессом выбора протоколом STP порта, исходя из стоимости портов
	1. Фиксируем вывод команды show spanning-tree до внесения изменений в стоимость поротов с некорневых коммутаторов S2 и S3.
	Для S2
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
	Для S3
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
	2. Изменяем стоимость корневого порта F0/4 на коммутаторе с самым высомим BID, то есть S3 (наибольший MAC-адрес)	
	S3(config)#int fa0/4
	S3(config-if)#spanning-tree cost 18
	3. Вывод команды show spanning-tree c некорневых коммутаторов S2 и S3.
	Для S2
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
	Для S3
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
	На сегменте между коммутаторами S2 и S3 назначенный и заблокированный порты поменялись местами.
	Так как на сегменте между коммутаторами S2 и S3 ни один из коммутаторов не является корневым. То выбор коммутатора, на котором будет назначенный порт зависит от стоимости от этих коммутаторов до корневого.
	Порт F0/2 коммутатора S3 с наименьшей стоимостью пути к корневому коммутатору (18) является назначенным портом.
	4. Удаление стоимость порта на интерфейсе F0/4 коммутатора S3
	S3(config)#int fa0/4
	S3(config-if)#no spanning-tree cost 
5. Наблюдение за процессом выбора протоколом STP порта, исходя из приоритета портов
	1. Включаем порты F0/1 и F0/3 на всех коммутаторах
	S1(config)#int range fa0/1,fa0/3
	S1(config-if-range)#no sh
	2. Просмотр данных протокола STP
		1. Для S1
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
		2. Для S2
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
		3. Для S3
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
		На некорневых коммутаторах в качестве корневого порта протоколом STP были выбраны следующие порты:
		На коммутаторе S2 порт F0/1; На коммуаторе S3 порт F0/3.
		Данные порты были выбраны протоколом STP в качестве корневых, так как порты отноятся к сегментам между корневым и некорневым коммутатороом (сегмент между S1 и S2; сегмент между S1 и S3).
		Если на таком сегменте присутсвует несколько линков, то приоритет отдается линку с наименьшим номером порта на корневом коммутаторе.
		Соответвенно, так как от S1 в сторону S2 два линка, подключенных в порты F0/1 и F0/2 на S1, то коммутатор S2 назначит корневым порт, соединенный с портом F0/1 на S1.
		Отсюда на S2 порт F0/1 - корневой, F0/2 - заблокированный
		Аналогиная ситуация между коммутаторами S1 и S3. Так как от S1 в сторону S3 два линка, подключенных в порты F0/3 и F0/4 на S1, то коммутатор S4 назначит корневым порт, соединенный с портом F0/3 на S1.
		Отсюда на S3 порт F0/3 - корневой, F0/4 - заблокированный
6. Обобщение порядка выбора портов протоколом STP
	1.	Какое значение протокол STP использует первым после выбора корневого моста, чтобы определить выбор порта? -- Cost (выбирается наименьшее)
	2.	Если первое значение на двух портах одинаково, какое следующее значение будет использовать протокол STP при выборе порта? -- Priority (выбирается наименьшее)
	3.	Если оба значения на двух портах равны, каким будет следующее значение, которое использует протокол STP при выборе порта? -- Номер порта на корневом коммутаторе (выбирается наименьшее)	
