# VLAN и маршрутизация между VLAN
## Физическая топология сети
![Физическая топология сети](https://github.com/Yabovbel/otus-networks/blob/6455dd8690afeb51f610f21408edb339e02ded49/labs/%5Blab_01%5D%20VLAN%20and%20inter-VLAN%20routing/pictures/otus-lab1.png "Физическая топология сети")
## Таблица адресации
Device	| Interface	| IP Address	| Subnet Mask	| Default Gateway
---	| ---		| ---		| ---		| ---
R1	| G0/0/1.3	| 192.168.3.1	| 255.255.255.0	| N/A
R1	| G0/0/1.4	| 192.168.4.1	| 255.255.255.0	| N/A
R1	| G0/0/1.8	| N/A		| N/A		| N/A
S1	| VLAN 3	| 192.168.3.11	| 255.255.255.0	| 192.168.3.1
S2	| VLAN 3	| 192.168.3.12	| 255.255.255.0	| 192.168.3.1
PC-A	| NIC		| 192.168.3.3	| 255.255.255.0	| 192.168.3.1
PC-B	| NIC		| 192.168.4.3	| 255.255.255.0	| 192.168.4.1

## Схема VLAN
![Схема VLAN](https://github.com/Yabovbel/otus-networks/blob/6455dd8690afeb51f610f21408edb339e02ded49/labs/%5Blab_01%5D%20VLAN%20and%20inter-VLAN%20routing/pictures/otus-lab1-1.png "Схема VLAN")
VLAN	| Name		| Interface Assigned
---	| ---		| ---
3	| Management	| S1: VLAN 3; S2: VLAN 3; S1: F0/6
4	| Operations	| S2: F0/18
7	| ParkingLot	| S1: F0/2-4, F0/7-24, G0/1-2; S2: F0/2-17, F0/19-24, G0/1-2 
8	| Native	| N/A

## Ход выполнения лабораторной работы
1. __Базовая конфигурация сетевых устройств (роутер, свитчи)__ 
 
	* Изменение имени командой `Switch(config)#hostname S1`  
	* Отключение DNS-lookup командой `S1(config)#no ip domain-lookup`  
	* Добавляем шифрованный пароль class для входа в привилегированный режим `S1(config)#enable secret class`  
	* Установка пароля cisco на консоль и виртуальные интерфейсы  
	```
	S1(config)#line vty 0 15
	S1(config-line)#password cisco
	S1(config-line)#login
	S1(config-line)#exit
	S1(config)#line console 0
	S1(config-line)#pass cisco
	S1(config-line)#login
	S1(config-line)#exit
	```  
	* Включение шифрования паролей в конфигурации `S1(config)#service password-encryption`  
	* Задаем приветсвенное сообщение пользователю `S1(config)#banner motd #unauthorized access is prohibited#`  
	* Вручную устанавливаем время  
	```
	S1#clock set 20:36:00 04 september 2022
	S1#show clock
	20:36:9.403 UTC Sun Sep 4 2022
	```  
	* Сохраняем текущую конфигурацию в NVRAM `S1#cop run sta`  
2. __Конфигурируем оконечные устройства в соответствии с адресацией__
3. __Настроиваем VLAN на свитчах__
	* Создаем и даем имена всем VLAN из таблицы адресации  
	```
	S1(config)#vlan 3
	S1(config-vlan)#name Management
	S1(config-vlan)#vl 4
	S1(config-vlan)#name Operations
	S1(config-vlan)#vl 7
	S1(config-vlan)#name ParkingLot
	S1(config-vlan)#vl 8
	S1(config-vlan)#name Native
	S1(config-vlan)#int vl 3
	```
	* Настраиваем адресацию на SVI для управления свитчем  
	```
	S1(config-if)#ip address 192.168.3.11 255.255.255.0
	S1(config-if)#no sh
	S1(config-if)#exit
	```
	* Настраиваем адрес шлюза для управления свитчем из удаленных подсетей через SVI.  
	`S1(config)#ip default-gateway 192.168.3.1`
	* Настраиваем VLAN на порты доступа. Неиспользуемые порты заводим в VLAN 7 и выключаем  
		1. Для S1  
		```
		S1(config)#int fa0/6
		S1(config-if)#switchport mode access 
		S1(config-if)#switchport ac vl 3
		S1(config-if)#exit 
		S1(config)#int range fa0/2-4,fa0/7-24,gi0/1-2
		S1(config-if-range)#switchport mode access 
		S1(config-if-range)#switchport ac vl 7
		S1(config-if-range)#shutdown 
		S1(config-if-range)#exit
		```
		2. Для S2  
		```
		S2(config)#int fa0/18
		S2(config-if)#switchport mode access 
		S2(config-if)#switchport access vl 4
		S2(config-if)#exit
		S2(config)#int range fa0/2-17,fa0/19-24,gi0/1-2
		S2(config-if-range)#switchport mode access 
		S2(config-if-range)#switchport access vlan 7
		S2(config-if-range)#shutdown
		S2(config-if-range)#exit
		```
	* Настраиваем транки на свитчах  
		1. Для S1  
		```
		S1(config)#int range fa0/1,fa0/5
		S1(config-if-range)#switchport mode trunk 
		S1(config-if-range)#switchport trunk native vlan 8
		S1(config-if-range)#switchport trunk allowed vlan 3,4,8
		S1(config-if-range)#exit
		```
		2. Для S2  
		```
		S2(config)#int range fa0/1
		S2(config-if-range)#switchport mode trunk 
		S2(config-if-range)#switchport trunk native vlan 8
		S2(config-if-range)#switchport trunk allowed vlan 3,4,8
		S2(config-if-range)#exit
		```
	* Проверяем натройку интерфейсов  
		1. Для S1  
		```
		S1#sh vl br
		VLAN Name                             Status    Ports
		---- -------------------------------- --------- -------------------------------
		1    default                          active    
		3    Management                       active    Fa0/6
		4    Operations                       active    
		7    ParkingLot                       active    Fa0/2, Fa0/3, Fa0/4, Fa0/7
								Fa0/8, Fa0/9, Fa0/10, Fa0/11
								Fa0/12, Fa0/13, Fa0/14, Fa0/15
								Fa0/16, Fa0/17, Fa0/18, Fa0/19
								Fa0/20, Fa0/21, Fa0/22, Fa0/23
								Fa0/24, Gig0/1, Gig0/2
		8    Native                           active    
		1002 fddi-default                     active    
		1003 token-ring-default               active    
		1004 fddinet-default                  active    
		1005 trnet-default                    active    
		```
		```
		S1#sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
		FastEthernet0/1        unassigned      YES manual up                    up 
		FastEthernet0/2        unassigned      YES manual administratively down down 
		FastEthernet0/3        unassigned      YES manual administratively down down 
		FastEthernet0/4        unassigned      YES manual administratively down down 
		FastEthernet0/5        unassigned      YES manual up                    up 
		FastEthernet0/6        unassigned      YES manual up                    up 
		FastEthernet0/7        unassigned      YES manual administratively down down 
		FastEthernet0/8        unassigned      YES manual administratively down down 
		FastEthernet0/9        unassigned      YES manual administratively down down 
		FastEthernet0/10       unassigned      YES manual administratively down down 
		FastEthernet0/11       unassigned      YES manual administratively down down 
		FastEthernet0/12       unassigned      YES manual administratively down down 
		FastEthernet0/13       unassigned      YES manual administratively down down 
		FastEthernet0/14       unassigned      YES manual administratively down down 
		FastEthernet0/15       unassigned      YES manual administratively down down 
		FastEthernet0/16       unassigned      YES manual administratively down down 
		FastEthernet0/17       unassigned      YES manual administratively down down 
		FastEthernet0/18       unassigned      YES manual administratively down down 
		FastEthernet0/19       unassigned      YES manual administratively down down 
		FastEthernet0/20       unassigned      YES manual administratively down down 
		FastEthernet0/21       unassigned      YES manual administratively down down 
		FastEthernet0/22       unassigned      YES manual administratively down down 
		FastEthernet0/23       unassigned      YES manual administratively down down 
		FastEthernet0/24       unassigned      YES manual administratively down down 
		GigabitEthernet0/1     unassigned      YES manual administratively down down 
		GigabitEthernet0/2     unassigned      YES manual administratively down down 
		Vlan1                  unassigned      YES manual administratively down down 
		Vlan3                  192.168.3.11    YES manual up                    up
		```
		```
		S1#sh int tr
		Port        Mode         Encapsulation  Status        Native vlan
		Fa0/1       on           802.1q         trunking      8
		Fa0/5       on           802.1q         trunking      8
		Port        Vlans allowed on trunk
		Fa0/1       3-4,8
		Fa0/5       3-4,8
		Port        Vlans allowed and active in management domain
		Fa0/1       3,4,8
		Fa0/5       3,4,8
		Port        Vlans in spanning tree forwarding state and not pruned
		Fa0/1       3,4,8
		Fa0/5       3,4,8
		```
		2. Для S2  
		```
		S2#sh vl br
		VLAN Name                             Status    Ports
		---- -------------------------------- --------- -------------------------------
		1    default                          active    
		3    Management                       active    
		4    Operations                       active    Fa0/18
		7    ParkingLot                       active    Fa0/2, Fa0/3, Fa0/4, Fa0/5
								Fa0/6, Fa0/7, Fa0/8, Fa0/9
								Fa0/10, Fa0/11, Fa0/12, Fa0/13
								Fa0/14, Fa0/15, Fa0/16, Fa0/17
								Fa0/19, Fa0/20, Fa0/21, Fa0/22
								Fa0/23, Fa0/24, Gig0/1, Gig0/2
		8    Native                           active    
		1002 fddi-default                     active    
		1003 token-ring-default               active    
		1004 fddinet-default                  active    
		1005 trnet-default                    active   
		```
		```
		S2#sh ip int br
		Interface              IP-Address      OK? Method Status                Protocol 
		FastEthernet0/1        unassigned      YES manual up                    up 
		FastEthernet0/2        unassigned      YES manual administratively down down 
		FastEthernet0/3        unassigned      YES manual administratively down down 
		FastEthernet0/4        unassigned      YES manual administratively down down 
		FastEthernet0/5        unassigned      YES manual administratively down down 
		FastEthernet0/6        unassigned      YES manual administratively down down 
		FastEthernet0/7        unassigned      YES manual administratively down down 
		FastEthernet0/8        unassigned      YES manual administratively down down 
		FastEthernet0/9        unassigned      YES manual administratively down down 
		FastEthernet0/10       unassigned      YES manual administratively down down 
		FastEthernet0/11       unassigned      YES manual administratively down down 
		FastEthernet0/12       unassigned      YES manual administratively down down 
		FastEthernet0/13       unassigned      YES manual administratively down down 
		FastEthernet0/14       unassigned      YES manual administratively down down 
		FastEthernet0/15       unassigned      YES manual administratively down down 
		FastEthernet0/16       unassigned      YES manual administratively down down 
		FastEthernet0/17       unassigned      YES manual administratively down down 
		FastEthernet0/18       unassigned      YES manual up                    up 
		FastEthernet0/19       unassigned      YES manual administratively down down 
		FastEthernet0/20       unassigned      YES manual administratively down down 
		FastEthernet0/21       unassigned      YES manual administratively down down 
		FastEthernet0/22       unassigned      YES manual administratively down down 
		FastEthernet0/23       unassigned      YES manual administratively down down 
		FastEthernet0/24       unassigned      YES manual administratively down down 
		GigabitEthernet0/1     unassigned      YES manual administratively down down 
		GigabitEthernet0/2     unassigned      YES manual administratively down down 
		Vlan1                  unassigned      YES manual administratively down down 
		Vlan3                  192.168.3.12    YES manual up                    up
		```
		```
		S2#sh int tr
		Port        Mode         Encapsulation  Status        Native vlan
		Fa0/1       on           802.1q         trunking      8
		Port        Vlans allowed on trunk
		Fa0/1       3-4,8
		Port        Vlans allowed and active in management domain
		Fa0/1       3,4,8
		Port        Vlans in spanning tree forwarding state and not pruned
		Fa0/1       3,4,8
		```
	* Для безопасности отключаем DTP на портах.  
	```
	S1(config)#int range fa0/1-24,gi0/1-2
	S1(config-if-range)#switchport nonegotiate 
	S1(config-if-range)#exit
	```
	* Для безопасности переводимо VTP  в режим Transparent  
	```
	S1(config)#vtp mode tr
	Setting device to VTP TRANSPARENT mode.
	S1#show vtp status 
	VTP Version capable             : 1 to 2
	VTP version running             : 2
	VTP Domain Name                 : 
	VTP Pruning Mode                : Disabled
	VTP Traps Generation            : Disabled
	Device ID                       : 00E0.B041.1A00
	Configuration last modified by 0.0.0.0 at 9-4-22 20:46:24
	Feature VLAN : 
	--------------
	VTP Operating Mode                : Transparent
	Maximum VLANs supported locally   : 255
	Number of existing VLANs          : 9
	Configuration Revision            : 0
	MD5 digest                        : 0x0A 0x71 0xD7 0x9E 0xE5 0xB3 0xB0 0x4C 
					    0x06 0xF3 0xCB 0x80 0x87 0xE8 0x8D 0x38
	```
	* Сохраняем текущую конфигурацию в NVRAM S1#cop run sta  
4. __Настраиваем терминацию VLAN на роутере__
	* Настраиваем sub-интерфейсы.  
	```
	R1(config)#int gi0/0/1.3
	R1(config-subif)#description Management
	R1(config-subif)#encapsulation dot1Q 3
	R1(config-subif)#ip address 192.168.3.1 255.255.255.0
	R1(config-subif)#int gi0/0/1.4
	R1(config-subif)#description Operations
	R1(config-subif)#encapsulation dot1Q 4
	R1(config-subif)#ip address 192.168.4.1 255.255.255.0
	R1(config-subif)#int gi0/0/1.8
	R1(config-subif)#description Native
	R1(config-subif)#encapsulation dot1Q 8 native 
	R1(config-subif)#exit
	```
	* Включаем физический интерфейс  
	```
	R1(config)#int gi0/0/1
	R1(config-if)#no sh
	```
	* Проверяем настройку sub-интерфейсов  
	```
	R1#show ip int br
	Interface              IP-Address      OK? Method Status                Protocol 
	GigabitEthernet0/0/0   unassigned      YES unset  administratively down down 
	GigabitEthernet0/0/1   unassigned      YES unset  up                    up 
	GigabitEthernet0/0/1.3 192.168.3.1     YES manual up                    up 
	GigabitEthernet0/0/1.4 192.168.4.1     YES manual up                    up 
	GigabitEthernet0/0/1.8 unassigned      YES unset  up                    up 
	GigabitEthernet0/0/2   unassigned      YES unset  administratively down down 
	Vlan1                  unassigned      YES unset  administratively down down
	```
5. __Настраиваем IPv4 на компьютерах согласно таблице адресации__
6. __Проверяем корректную работу сети с помощью команды ping с компьютера PC-A__
	* Ping from PC-A to its default gateway.  
	```
	C:\>ping 192.168.3.1

	Pinging 192.168.3.1 with 32 bytes of data:

	Reply from 192.168.3.1: bytes=32 time<1ms TTL=255
	Reply from 192.168.3.1: bytes=32 time<1ms TTL=255
	Reply from 192.168.3.1: bytes=32 time<1ms TTL=255
	Reply from 192.168.3.1: bytes=32 time<1ms TTL=255

	Ping statistics for 192.168.3.1:
		Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
	Approximate round trip times in milli-seconds:
		Minimum = 0ms, Maximum = 0ms, Average = 0ms
	```
	* Ping from PC-A to PC-B  
	```
	C:\>ping 192.168.4.3

	Pinging 192.168.4.3 with 32 bytes of data:

	Request timed out.
	Reply from 192.168.4.3: bytes=32 time=7ms TTL=127
	Reply from 192.168.4.3: bytes=32 time<1ms TTL=127
	Reply from 192.168.4.3: bytes=32 time<1ms TTL=127

	Ping statistics for 192.168.4.3:
		Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
	Approximate round trip times in milli-seconds:
		Minimum = 0ms, Maximum = 7ms, Average = 2ms
	```
	* Ping from PC-A to S2  
	```
	C:\>ping 192.168.3.12

	Pinging 192.168.3.12 with 32 bytes of data:

	Request timed out.
	Reply from 192.168.3.12: bytes=32 time<1ms TTL=255
	Reply from 192.168.3.12: bytes=32 time<1ms TTL=255
	Reply from 192.168.3.12: bytes=32 time<1ms TTL=255

	Ping statistics for 192.168.3.12:
		Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
	Approximate round trip times in milli-seconds:
		Minimum = 0ms, Maximum = 0ms, Average = 0ms
	```
