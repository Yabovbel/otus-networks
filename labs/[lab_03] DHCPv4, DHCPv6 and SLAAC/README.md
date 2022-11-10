# DHCPv4/v6 и SLAAC 
# PART 1.  Базовая настройка сетевых устройств. Настройка VLAN. Настройка IPv4 адресации. Создание DHCPv4 сервера.
## Физическая топология сети



1. Разработка плана IP адресации для сети
	Название подсети | Адрес подсети | Маска подсети | Требуемое количество хостов | Количество адресов хостов
	Subnet A	| 192.168.1.0	| 255.255.255.192(/26) | 58 | 62 (192.168.1.1-192.168.1.62)
	Subnet B	| 192.168.1.64	| 255.255.255.224(/27) | 28 | 30 (192.168.1.65-192.168.1.94)
	Subnet C	| 192.168.1.96	| 255.255.255.240(/28) | 12 | 14 (192.168.1.97-192.168.1.110)
2. Настройка базовой конфигурации на роутерах R1, R2 и коммуаторах S1 и S2
	1. Задаем имя устройства командой Router(config)#hostname R1
	2. Отключаем domain-lookup чтобы маршрутизатор не пытался транслировать неправильно введенные команды, как имена хостов. Для этого выполняем команду R1(config)#no ip domain-lookup
	3. Задаем для доступа к привелигерованному режиму шифрованный пароль *class* командой R1(config)#enab sec class
	4. Задаем пароль *cisco* для доступа к консоли и виртуальному интерфейсу. Также просим IOS восстановливать набранную команду, если ввод был прерван системным сообoением из syslog 
		R1(config)#line vty 0 15
		R1(config-line)#password cisco
		R1(config-line)#login
		R1(config-line)#logging synchronous 
		R1(config-line)#exit
		R1(config)#line con 0
		R1(config-line)#password cisco
		R1(config-line)#login 
		R1(config-line)#logging synchronous 
		R1(config-line)#exit
	5. Шифруем открытые пароли, заданные в конфигурации R1(config)#service password-encryption
	6. Задаем приветственное сообщение для вошедших пользователей R1(config)#banner motd #Unauthorized access is prohibited#
	7. Задаем дату и время для более удобного анализа логов.
		R1#clock set 15:19:00 12 october 2022
		R1#sh clo
		15:19:5.827 UTC Wed Oct 12 2022
	8. Сохраняем текущую кофигурацию в NVRAM
		R1#cop run sta
		Destination filename [startup-config]? 
		Building configuration...
		[OK]
3. Настройка субинтерфейсов на роутере R1
	1. Создаем субинтерфейс 100 на порту gi0/0/1 и назначаем ему VLAN 100
		R1(config)#interface gi0/0/1.100
		R1(config-subif)#encapsulation dot1Q 100
	2. Добавлем описание к субинтерфейсу командой R1(config-subif)#description ClientsVLAN
	3. Добавлем IP адрес согласно таблице адресации командой R1(config-subif)#ip address 192.168.1.1 255.255.255.192
    4. Аналогично настраиваем субинтерфейс для VLAN 200
		R1(config-subif)#int gi0/0/1.200
		R1(config-subif)#encapsulation dot1Q 200
		R1(config-subif)#description ManagementVLAN
		R1(config-subif)#ip address 192.168.1.65 255.255.255.224
	5. Добавляем субинтерфейс для nаtive VLAN 1000. IP адрес интерфейсу не задаем. 
		R1(config-subif)#int gi0/0/1.1000
		R1(config-subif)#encapsulation dot1Q 1000 native 
		R1(config-subif)#description NativeVLAN
	6. Проверяем корректность настройки адресов на интерфейсах. Вывод команды show ip interface brief
		R1(config-subif)#do sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
		GigabitEthernet0/0/0   unassigned      YES unset  administratively down down 
		GigabitEthernet0/0/1   unassigned      YES unset  administratively down down 
		GigabitEthernet0/0/1.100192.168.1.1     YES manual administratively down down 
		GigabitEthernet0/0/1.200192.168.1.65    YES manual administratively down down 
		GigabitEthernet0/0/1.1000unassigned      YES unset  administratively down down 
		Vlan1                  unassigned      YES unset  administratively down down
	7. Добавляем описание на порт gi0/0/1 и включаем его
		R1(config-subif)#int gi0/0/1
		R1(config-if)#description trunkToS1
		R1(config-if)#no sh
4. Настройка остальных интерфейсов на роутерах R1 и R2
	1. На роутере R2 задаем IP адреса согласно таблице адресации интерфейсам gi0/0/0 и gi0/0/1. Добавляем описание, включаем.
		R2(config)#int gi0/0/1
		R2(config-if)#description linkToS2
		R2(config-if)#ip address 192.168.1.97 255.255.255.240
		R2(config-if)#no sh
		R2(config-if)#int gi0/0/0
		R2(config-if)#description linkToR1
		R2(config-if)#ip address 10.0.0.2 255.255.255.252
		R2(config-if)#no sh
	2. На роутере R1 задаем IP адрес согласно таблице адресации интерфейсу gi0/0/0. Добавляем описание, включаем.
		R1(config)#int gi0/0/0
		R1(config-if)#description linkToR2
		R1(config-if)#ip address 10.0.0.1 255.255.255.252
		R1(config-if)#no sh
	3. На роутере R2 задаем маршруты до подсетей 192.168.1.1/26 и 192.168.1.64/27, которые за роутером R1.
		В задании сказано прописать дефолтные (в нули) маршруты на интерфейс соседнего роутера, но на мой взгляд,
		данная конфигурация будет некорректно работать для пакетов чьи адреса назначения не будет найдены в более 
		точных маршрутах, чем в дефолтном (в данном случае это все подсети, кроме локально подключенных к роутерам). 
		Такие пакеты будет зацикливаться :(.
		Поэтому прописываю маршруты до конкретых подсетей.
			Мальчик сказал маме: “Я хочу кушать”. Мама отправила его к папе.
			Мальчик сказал папе: “Я хочу кушать”. Папа отправил его к маме.
			Мальчик сказал маме: “Я хочу кушать”. Мама отправила его к папе.
			И бегал так мальчик, пока в один момент не упал.
			Что случилось с мальчиком? TTL кончился.
		R2(config)#ip route 192.168.1.0 255.255.255.192 10.0.0.1
		R2(config)#ip route 192.168.1.64 255.255.255.224 10.0.0.1
	4. Аналогично настраиваем маршруты на роутере R1 до подсети 192.168.1.96/28
		R1(config)#ip route 192.168.1.96 255.255.255.240 10.0.0.2
	5. Проверяем корректность IP адресов на интерфейсах командой
		На R1
			R1#sh ip int br
			Interface              IP-Address      OK? Method Status                Protocol 
			GigabitEthernet0/0/0   10.0.0.1        YES manual up                    up 
			GigabitEthernet0/0/1   unassigned      YES unset  up                    up 
			GigabitEthernet0/0/1.100192.168.1.1     YES manual up                    up 
			GigabitEthernet0/0/1.200192.168.1.65    YES manual up                    up 
			GigabitEthernet0/0/1.1000unassigned      YES unset  up                    up 
			Vlan1                  unassigned      YES unset  administratively down down
		На R2
			R2#sh ip int br
			Interface              IP-Address      OK? Method Status                Protocol 
			GigabitEthernet0/0/0   10.0.0.2        YES manual up                    up 
			GigabitEthernet0/0/1   192.168.1.97    YES manual up                    up 
			Vlan1                  unassigned      YES unset  administratively down down
	6. Проверяем корректность настройки статических маршрутов командой
		На R1
			R1#sh ip route 
			...
			Gateway of last resort is not set

				 10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
			C       10.0.0.0/30 is directly connected, GigabitEthernet0/0/0
			L       10.0.0.1/32 is directly connected, GigabitEthernet0/0/0
				 192.168.1.0/24 is variably subnetted, 5 subnets, 4 masks
			C       192.168.1.0/26 is directly connected, GigabitEthernet0/0/1.100
			L       192.168.1.1/32 is directly connected, GigabitEthernet0/0/1.100
			C       192.168.1.64/27 is directly connected, GigabitEthernet0/0/1.200
			L       192.168.1.65/32 is directly connected, GigabitEthernet0/0/1.200
			S       192.168.1.96/28 [1/0] via 10.0.0.2
		На R2
			R2#sh ip route 
			...
			Gateway of last resort is not set

				 10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
			C       10.0.0.0/30 is directly connected, GigabitEthernet0/0/0
			L       10.0.0.2/32 is directly connected, GigabitEthernet0/0/0
				 192.168.1.0/24 is variably subnetted, 4 subnets, 4 masks
			S       192.168.1.0/26 [1/0] via 10.0.0.1
			S       192.168.1.64/27 [1/0] via 10.0.0.1
			C       192.168.1.96/28 is directly connected, GigabitEthernet0/0/1
			L       192.168.1.97/32 is directly connected, GigabitEthernet0/0/1
	7. Сохраняем текущую конфигурации
5. Настройка VLAN на коммутаторе S1
	1. Создание и присваивание имен VLAN.
		S1(config)#vlan 100
		S1(config-vlan)#name Clients
		S1(config-vlan)#vlan 200
		S1(config-vlan)#name Management
		S1(config-vlan)#vlan 999
		S1(config-vlan)#name ParkingLot
		S1(config-vlan)#vlan 1000
		S1(config-vlan)#name Native
	2. На коммутаторе S1 cоздаем SVI интерфейс управления на VLAN 200.
		S1(config)#int vl 200
		S1(config-if)#ip address 192.168.1.66 255.255.255.224
		S1(config-if)#no sh
	3. Задаем IP адрес шлюза командой S1(config)#ip default-gateway 192.168.1.65
6. Настройка SVI и портов на коммутаторе S2
	1. На коммутаторе S2 прописываем IP адрес для SVI интерфейса VLAN 1 (по умолчанию он уже создан) и включаем его
		S2(config)#int vl 1
		S2(config-if)#ip address 192.168.1.98 255.255.255.240
		S2(config-if)#no sh
	2. Задаем IP адрес шлюза командойS2(config)#ip default-gateway 192.168.1.97
	3. Для безопасности отключаем неиспользуемые порты на коммутаторе командой
		S2(config)#int range fa0/1-4,fa0/6-17,fa0/19-24,gi0/1-2
		S2(config-if-range)#sh
7. Добавление портов на коммутаторе S1 в соответвующие VLAN.
	1. Добавляем порты на коммутаторе S1 в VLAN в соответвии с таблицей VLAN.
		S1(config)#int fa0/6
		S1(config-if)#switchport mode access 
		S1(config-if)#switchport access vlan 100
	2. Все неиспользуемые порты добавляем в 999 VLAN (ParkingLot) и выключаем их
		S1(config)#int range fa0/1-4,fa0/7-24,gi0/1-2
		S1(config-if-range)#switchport mode access 
		S1(config-if-range)#switchport access vlan 999
		S1(config-if-range)#sh
	3. Проверяем корректность присваивания VLAN портам коммутатора
		S1#sh vl br
		VLAN Name                             Status    Ports
		---- -------------------------------- --------- -------------------------------
		1    default                          active    Fa0/5
		100  Clients                          active    Fa0/6
		200  Management                       active    
		999  ParkingLot                       active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
														Fa0/7, Fa0/8, Fa0/9, Fa0/10
														Fa0/11, Fa0/12, Fa0/13, Fa0/14
														Fa0/15, Fa0/16, Fa0/17, Fa0/18
														Fa0/19, Fa0/20, Fa0/21, Fa0/22
														Fa0/23, Fa0/24, Gig0/1, Gig0/2
		1000 Native                           active    
		1002 fddi-default                     active    
		1003 token-ring-default               active    
		1004 fddinet-default                  active    
		1005 trnet-default                    active   
		На данный момент порт F0/5 находится по умолчнию в VLAN 1, так как для него никакие настройки VLAN не были заданы. Нам нужно это исправить, указав порт F0/5 как транковый.
	4. Конфигурируем порт F0/5 на коммутаторе S1 в качестве транкового
		S1(config)#int fa0/5
		S1(config-if)#switchport mode tr
		S1(config-if)#switchport trunk native vl 1000
		S1(config-if)#switchport trunk allowed vlan 100,200,1000
	5. Проверяем настройку транковых интерфейсов командой
		S1#sh int tr
		Port        Mode         Encapsulation  Status        Native vlan
		Fa0/5       on           802.1q         trunking      1000

		Port        Vlans allowed on trunk
		Fa0/5       100,200,1000

		Port        Vlans allowed and active in management domain
		Fa0/5       100,200,1000

		Port        Vlans in spanning tree forwarding state and not pruned
		Fa0/5       100,200,1000
	6. Сохраняем текущую конфигурацию в NVRAM с помощью команды S1#cop run sta
	7. Теперь, после настройки DHCPv4 на роутере R1 компьютер PC-A должен получить адрес из подсети 192.168.1.0/26 за исключением адреса шлюза 192.168.1.1
8. Настройка DHCPv4 серверов на роутере R1 для подсетей Subnet A (192.168.1.0/26) и Subnet С (192.168.1.96/28)
	1. Для двух подсетей исключаем первые 5 адресов хостов из выдачи DHCP сервера
		R1(config)#ip dhcp excluded-address 192.168.1.1 192.168.1.5
		R1(config)#ip dhcp excluded-address 192.168.1.97 192.168.1.101
	2. Создаем DHCP пул для Subnet A (192.168.1.0/26) 
		R1(config)#ip dhcp pool SUBNET_A
		R1(dhcp-config)#network 192.168.1.0 255.255.255.192
		R1(dhcp-config)#domain-name ccna-lab.com
		R1(dhcp-config)#default-router 192.168.1.1
		R1(dhcp-config)#lease 2 12 30
	3. Создаем DHCP пул для Subnet C (192.168.1.96/28)
		R1(config)#ip dhcp pool R2_Client_LAN
		R1(dhcp-config)#network 192.168.1.96 255.255.255.240
		R1(dhcp-config)#domain-name ccna-lab.com
		R1(dhcp-config)#default-router 192.168.1.97
		R1(dhcp-config)#lease 2 12 30
	4. Сохраняем текущую конфигурацию в NVRAM с помощью команды S1#cop run sta
	5. Проверяем конфигурацию и работоспособность DHCP сервера на роутере R1
		a. Просмотр деталей DHCP пулов выполняем командой
			R1#sh ip dhcp pool

			Pool SUBNET_A :
			 Utilization mark (high/low)    : 100 / 0
			 Subnet size (first/next)       : 0 / 0 
			 Total addresses                : 62
			 Leased addresses               : 0
			 Excluded addresses             : 2
			 Pending event                  : none

			 1 subnet is currently in the pool
			 Current index        IP address range                    Leased/Excluded/Total
			 192.168.1.1          192.168.1.1      - 192.168.1.62      0    / 2     / 62

			Pool R2_Client_LAN :
			 Utilization mark (high/low)    : 100 / 0
			 Subnet size (first/next)       : 0 / 0 
			 Total addresses                : 14
			 Leased addresses               : 0
			 Excluded addresses             : 2
			 Pending event                  : none

			 1 subnet is currently in the pool
			 Current index        IP address range                    Leased/Excluded/Total
			 192.168.1.97         192.168.1.97     - 192.168.1.110     0    / 2     / 14
		b. Просмотр выданных DHCP сервером клиентам адресов
			R1#show ip dhcp binding 
			IP address       Client-ID/              Lease expiration        Type
							 Hardware address
			192.168.1.6      0001.C7B4.8173           --                     Automatic
		с.  Выполнить команду show ip dhcp server statistics в packet tracer 8.0.0 не получилось
	6. Проверяем получение IP адреса компьтером PC-A
		a. Вывод команды ipconfig
			C:\>ipconfig
			FastEthernet0 Connection:(default port)
			   Connection-specific DNS Suffix..: ccna-lab.com
			   Link-local IPv6 Address.........: FE80::201:C7FF:FEB4:8173
			   IPv6 Address....................: ::
			   IPv4 Address....................: 192.168.1.6
			   Subnet Mask.....................: 255.255.255.192
			   Default Gateway.................: ::
												 192.168.1.1
			...
		b. Выполнение ping-теста до роутера R1
			C:\>ping 192.168.1.1

			Pinging 192.168.1.1 with 32 bytes of data:

			Reply from 192.168.1.1: bytes=32 time<1ms TTL=255
			Reply from 192.168.1.1: bytes=32 time<1ms TTL=255
			Reply from 192.168.1.1: bytes=32 time<1ms TTL=255
			Reply from 192.168.1.1: bytes=32 time<1ms TTL=255

			Ping statistics for 192.168.1.1:
				Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
			Approximate round trip times in milli-seconds:
				Minimum = 0ms, Maximum = 0ms, Average = 0ms
9. Настройка DHCP Relay на роутере R2.
	1. Настройка helper на интерфейсе G0/0/1 роутера R2
		R2(config)#int gi0/0/1
		R2(config-if)#ip helper-address 10.0.0.1
	2. Сохраняем текущую конфигурацию в NVRAM с помощью команды S1#cop run sta
	3. Проверяем получение IP адреса компьтером PC-A
		a. Вывод команды ipconfig
			C:\>ipconfig
			FastEthernet0 Connection:(default port)
			   Connection-specific DNS Suffix..: ccna-lab.com
			   Link-local IPv6 Address.........: FE80::20C:CFFF:FE00:BAA1
			   IPv6 Address....................: ::
			   IPv4 Address....................: 192.168.1.102
			   Subnet Mask.....................: 255.255.255.240
			   Default Gateway.................: ::
												 192.168.1.97
			...
		b. Выполнение ping-теста до роутера R2
			C:\>ping 192.168.1.97

			Pinging 192.168.1.97 with 32 bytes of data:

			Reply from 192.168.1.97: bytes=32 time<1ms TTL=255
			Reply from 192.168.1.97: bytes=32 time<1ms TTL=255
			Reply from 192.168.1.97: bytes=32 time<1ms TTL=255
			Reply from 192.168.1.97: bytes=32 time<1ms TTL=255

			Ping statistics for 192.168.1.97:
				Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
			Approximate round trip times in milli-seconds:
				Minimum = 0ms, Maximum = 0ms, Average = 0ms
	4. Просмотр выданных DHCP сервером адресов после подключения компьютера PC-B
		R1#sh ip dhcp binding 
		IP address       Client-ID/              Lease expiration        Type
						 Hardware address
		192.168.1.6      0001.C7B4.8173           --                     Automatic
		192.168.1.102    000C.CF00.BAA1           --                     Automatic
	5. Выполнить команду show ip dhcp server statistics в packet tracer 8.0.0 не получилось

II. PART 2. Базовая настройка сетевых устройств. Настройка IPv6 адресации. Создание DHCPv6 сервера.
1. Настройка базовой конфигурации на роутерах R1, R2 и коммуаторах S1 и S2
	1. Выполняем действия, описанные в пункте 2 первой части лабораторной работы [ссылка]
	2. Выключаем неиспользуемые порты на коммутатораях S1 и S2 в целях безопасности.
		Для S1
			S1(config)#int range fa0/1-4,fa0/7-24,gi0/1-2
			S1(config-if-range)#sh
		Для S2
			S2(config)#int range fa0/1-4,fa0/6-17,fa0/19-24,gi0/1-2
			S2(config-if-range)#sh
2. Настройка интерфейсов и маршрутизации для IPv6 на роутерах R1 и R2
	1. Активируем IPv6 маршрутизацию командой R1(config)#ipv6 unicast-routing
	2. Настраиваем IPv6 адреса на интерфейсах роуетров R1 и R2 в соответствии с таблицей адресации
		Для R1
			R1(config)#int g0/0/0
			R1(config-if)#ipv6 address 2001:db8:acad:2::1/64
			R1(config-if)#ipv6 address fe80::1 link-local 
			R1(config-if)#no sh
			R1(config-if)#int gi0/0/1
			R1(config-if)#ipv6 address 2001:db8:acad:1::1/64
			R1(config-if)#ipv6 address fe80::1 link-local 
			R1(config-if)#no sh
		Для R2
			R2(config)#int g0/0/0
			R2(config-if)#ipv6 address 2001:db8:acad:2::2/64
			R2(config-if)#ipv6 address fe80::2 link-local 
			R2(config-if)#no sh
			R2(config-if)#int g0/0/1
			R2(config-if)#ipv6 address 2001:db8:acad:3::1/64
			R2(config-if)#ipv6 address fe80::1 link-local 
			R2(config-if)#no sh
	3. Настраиваем статические маршруты на роутерах R1 и R2.
		Всвязи с проблемой, описанной в пункте 4.3 первой части лабораторной работы [ссылка], вместо дефолтных прописываю маршруты до конкретных подсетей
		Для R1
			R1(config)#ipv6 route 2001:db8:acad:3::1/64 2001:db8:acad:2::2 
		Для R2
			R2(config)#ipv6 route 2001:db8:acad:1::1/64 2001:db8:acad:2::1
	5. Проверка корректности настройки IPv6 адресов и статических маршрутов
		a. Вывод команды show ipv6 int br
			Для R1
				R1#sh ipv6 int br
				GigabitEthernet0/0/0       [up/up]
					FE80::1
					2001:DB8:ACAD:2::1
				GigabitEthernet0/0/1       [up/up]
					FE80::1
					2001:DB8:ACAD:1::1
				Vlan1                      [administratively down/down]
					unassigned
			Для R2
				R2#sh ipv6 int br
				GigabitEthernet0/0/0       [up/up]
					FE80::2
					2001:DB8:ACAD:2::2
				GigabitEthernet0/0/1       [up/up]
					FE80::1
					2001:DB8:ACAD:3::1
				Vlan1                      [administratively down/down]
					unassigned	
		b. Вывод команды show ipv6 route
			Для R1
				R1#sh ipv6 route 
				IPv6 Routing Table - 6 entries
				...
				C   2001:DB8:ACAD:1::/64 [0/0]
					 via GigabitEthernet0/0/1, directly connected
				L   2001:DB8:ACAD:1::1/128 [0/0]
					 via GigabitEthernet0/0/1, receive
				C   2001:DB8:ACAD:2::/64 [0/0]
					 via GigabitEthernet0/0/0, directly connected
				L   2001:DB8:ACAD:2::1/128 [0/0]
					 via GigabitEthernet0/0/0, receive
				S   2001:DB8:ACAD:3::/64 [1/0]
					 via 2001:DB8:ACAD:2::2
				L   FF00::/8 [0/0]
					 via Null0, receive
			Для R2
				R2#sh ipv6 route 
				IPv6 Routing Table - 6 entries
				...
				S   2001:DB8:ACAD:1::/64 [1/0]
					 via 2001:DB8:ACAD:2::1
				C   2001:DB8:ACAD:2::/64 [0/0]
					 via GigabitEthernet0/0/0, directly connected
				L   2001:DB8:ACAD:2::2/128 [0/0]
					 via GigabitEthernet0/0/0, receive
				C   2001:DB8:ACAD:3::/64 [0/0]
					 via GigabitEthernet0/0/1, directly connected
				L   2001:DB8:ACAD:3::1/128 [0/0]
					 via GigabitEthernet0/0/1, receive
				L   FF00::/8 [0/0]
					 via Null0, receive
	4. Проверка корректности настройки IPv6 эхо-запросом с роутера R1 на адрес интерфейса G0/0/1 роутера R2.
		R1#ping 2001:db8:acad:3::1
		Type escape sequence to abort.
		Sending 5, 100-byte ICMP Echos to 2001:db8:acad:3::1, timeout is 2 seconds:
		!!!!!
		Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms
	5. Сохраняем текущую конфигурацию в NVRAM с помощью команды S1#cop run sta
3. Проверка работоспособности SLAAC
	1. Выполняем команду ipconfig /all с компьютера PC-A
		C:\>ipconfig /all

		FastEthernet0 Connection:(default port)

		   Connection-specific DNS Suffix..: 
		   Physical Address................: 0001.C7B4.8173
		   Link-local IPv6 Address.........: FE80::201:C7FF:FEB4:8173
		   IPv6 Address....................: 2001:DB8:ACAD:1:201:C7FF:FEB4:8173
		   IPv4 Address....................: 0.0.0.0
		   Subnet Mask.....................: 0.0.0.0
		   Default Gateway.................: FE80::1
											 0.0.0.0
		   DHCP Servers....................: 0.0.0.0
		   DHCPv6 IAID.....................: 
		   DHCPv6 Client DUID..............: 00-01-00-01-A6-37-40-2C-00-01-C7-B4-81-73
		   DNS Servers.....................: ::
											 0.0.0.0
		...
	2. Идентификатор хоста был сформирован на основе MAC- адреса сетевого интерфейса компьютера PC-A по алгоритму EUI-64
		MAC-адрес сетевого интерфейса компьютера PC-A: 0001.C7B4.8173
		[картинка1]
4. Настройка и проверка stateless DHCPv6 сервера на роутере R1
	На данный момент компьютер не получает от DHCPv6 сервера unicast адреса DNS серверов.
	1. Создаем DHCP пул с именем R1-STATELESS. В нем указываем адрес DNS сервера и имя домена.
		R1(config)#ipv6 dhcp pool R1-STATELESS
		R1(config-dhcpv6)#dns-server 2001:db8:acad::254
		R1(config-dhcpv6)#domain-name STATELESS.com
	2. Привязываем пул DHCPv6 сервера к интерфейсу G0/0/1. Устанавливаем флаг "O" в единицу, который указывается,
	что адреса получаются через SLAAC, а прочая информация (адреса DNS) получаются от DHCP сервера.
		R1(config)#int g0/0/1
		R1(config-if)#ipv6 nd other-config-flag 
		R1(config-if)#ipv6 dhcp server R1-STATELESS
	3. Сохраняем текущую конфигурацию в NVRAM с помощью команды S1#cop run sta
	4. Перезагружаем компьютер PC-A и выполняем команду ipconfig /all
		C:\>ipconfig /all
		FastEthernet0 Connection:(default port)
		   Connection-specific DNS Suffix..: 
		   Physical Address................: 0001.C7B4.8173
		   Link-local IPv6 Address.........: FE80::201:C7FF:FEB4:8173
		   IPv6 Address....................: ::
		   IPv4 Address....................: 0.0.0.0
		   Subnet Mask.....................: 0.0.0.0
		   Default Gateway.................: FE80::1
											 0.0.0.0
		   DHCP Servers....................: 0.0.0.0
		   DHCPv6 IAID.....................: 
		   DHCPv6 Client DUID..............: 00-01-00-01-A6-37-40-2C-00-01-C7-B4-81-73
		   DNS Servers.....................: 2001:DB8:ACAD::254
											 0.0.0.0
		Наблюдаем, что компьютер получил по DHCPv6 адрес DNS-сервера.
	5. Проверяем прохождение ping запроса на адрес интерфейса G0/0/1 на роутере R2
		C:\>ping 2001:DB8:ACAD:3::1

		Pinging 2001:DB8:ACAD:3::1 with 32 bytes of data:

		Reply from 2001:DB8:ACAD:3::1: bytes=32 time<1ms TTL=254
		Reply from 2001:DB8:ACAD:3::1: bytes=32 time<1ms TTL=254
		Reply from 2001:DB8:ACAD:3::1: bytes=32 time<1ms TTL=254
		Reply from 2001:DB8:ACAD:3::1: bytes=32 time<1ms TTL=254

		Ping statistics for 2001:DB8:ACAD:3::1:
			Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
		Approximate round trip times in milli-seconds:
			Minimum = 0ms, Maximum = 0ms, Average = 0ms
5. Конфигурирование stateful DHCPv6 сервера на роутере R1.
	1. Создаем DHCP пул с именем R2-STATEFUL. В нем указываем префикс адреса диапазона, из которого будут выдываться IP-адреса (2001:db8:acad:3:aaa::/80).
	Также указывается адрес DNS сервера и имя домена.
		R1(config)# ipv6 dhcp pool R2-STATEFUL
		R1(config-dhcp)# address prefix 2001:db8:acad:3:aaa::/80
		R1(config-dhcp)# dns-server 2001:db8:acad:3::254
		R1(config-dhcp)# domain-name STATEFUL.com
	2. Привязываем пул DHCPv6 сервера к интерфейсу G0/0/0, чтобы он был доступен для DHCP Relay с роутера R2.
		R1(config)# interface g0/0/0
		R1(config-if)# ipv6 dhcp server R2-STATEFUL
6. Настройка и проверка DHCPv6 Relay на роутере R2.
	1. Выполняем команду ipconfig /all с компьютера PC-B
		C:\>ipconfig /all
		FastEthernet0 Connection:(default port)
		   Connection-specific DNS Suffix..: 
		   Physical Address................: 000C.CF00.BAA1
		   Link-local IPv6 Address.........: FE80::20C:CFFF:FE00:BAA1
		   IPv6 Address....................: 2001:DB8:ACAD:3:20C:CFFF:FE00:BAA1
		   IPv4 Address....................: 0.0.0.0
		   Subnet Mask.....................: 0.0.0.0
		   Default Gateway.................: FE80::1
											 0.0.0.0
		   DHCP Servers....................: 0.0.0.0
		   DHCPv6 IAID.....................: 
		   DHCPv6 Client DUID..............: 00-01-00-01-45-08-A1-47-00-0C-CF-00-BA-A1
		   DNS Servers.....................: ::
											 0.0.0.0
		Префикс 2001:db8:acad:3:: был получен с помощью SLAAC, так как на интерфейсе роутера данного широковещательного домена настроен IP адрес 2001:DB8:ACAD:3::1
	--------------------------------------------------------
	Оставшаяся часть лабораторной работы выполнялась в EVE-NG, так как в Packet Tracer 8.0 недоступен функционал DHCPv6 relay. В качестве DHCPv6 клиента использован образ Cisco IOL L2
	--------------------------------------------------------
	2. Настройка DHCP Relay на роутере R2 для интерфейса G0/0/1.
		a. Для интерфейса G0/0/1 указываем "M" флаг и указываем адрес роутера R1 в качестве назначения для пересылки DHCPv6 пакетов
		R2(config)# interface g0/0/1
		R2(config-if)# ipv6 nd managed-config-flag
		R2(config-if)# ipv6 dhcp relay destination 2001:db8:acad:2::1 g0/0/0
	3. Проверка получения конфигурации по DHCPv6 клиентом
		SW6(config)#int vl 1
		SW6(config-if)#ipv6 enable
		SW6(config-if)#ipv6 address dhcp
		SW6#sh ipv6 dhcp int	
		Vlan1 is in client mode
		  Prefix State is IDLE
		  Address State is OPEN
		  Renew for address will be sent in 11:56:50
		  List of known servers:
			Reachable via address: FE80::1
			DUID: 00030001AABBCC001000
			Preference: 0
			Configuration parameters:
			  IA NA: IA ID 0x00070001, T1 43200, T2 69120
				Address: 2001:DB8:ACAD:3:AAA:FBBE:4465:B070/128
						preferred lifetime 86400, valid lifetime 172800
						expires at Oct 29 2022 12:02 AM (172611 seconds)
			  DNS server: 2001:DB8:ACAD:3::254
			  Domain name: STATEFUL.com
			  Information refresh time: 0
		  Prefix Rapid-Commit: disabled
		  Address Rapid-Commit: disabled
