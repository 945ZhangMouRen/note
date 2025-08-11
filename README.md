# note
Record problems in works

CE交换机维护宝典（面向二线）
一、文档目的：
	总结现网的常见问题故障，针对故障场景定界为配置还是软硬件BUG造成，提供场景下采集信息指导。

二、常见故障场景总结：
1、	重启类场景问题
2、	堆叠场景问题，包含堆叠无法组建、堆叠分裂等
3、	M-LAG场景问题，包含M-LAG场景下转发定界方法
4、	ACL场景问题，包含ACL无法下发、ACL不生效
5、	端口场景问题，包含端口闪断，端口无法UP
6、	二层防环问题，包含MSTP、VBST场景下问题
7、	三层转发异常，包含三层转发问题的一般判断方法
8、	VXLAN场景问题，包含VXLAN场景下统计方法

快速查询---最常见故障日志收集方法
a)	logfile目录下复位时间段的log.log 和diag.log以及压缩包日志
b)	display diag回显
c)	诊断视图下敲coll dia info在根目录下生成的diagnostic_information.zip文件，把文件FTP取出来
  
  KPI文件收集：
1)	V2R5C10 : KPI文件集成到diagnostic_information.zip文件，取它即可
2)	V2R19C10开始： 在根目录下KPISTAT文件夹内，以设备当前时间保存。
 
三、重启类场景问题
重启类场景问题一般可以分为两种，一种为重启后已注册，一种为频繁重启一直未注册，针对第二种情况在排除FD海力士单板已知预警后，大概率需要做RMA更换处理。对于第一种则需要按照下面过程处理
https://support.huawei.com/enterprise/zh/bulletins-product/ENEWS2000006068   --海力士FD单板预警，此预警建议二线兄弟都能知悉。

1、	display device board reset all 查看单板重启原因，回显如下：
其中的Reason会说明对应的情况，一般如下：
Board register ： 单板注册，重启后重新注册的记录
Board cold reset(COLD Reset) ： 冷复位，掉电造成的异常复位，需要排查供电
Reboot board from command.(CPU Reset)  ： 命令行主动操作复位，排查客户操作
LAN Switch parity/ecc error, and reset board.(CPU Reset) ： 软失效复位

<CE6851HI_147.65>display device  board reset  all                               
----------------------------------------------------------                      
Board 1 reset information:                                                      
-- 1. DATE:2020-10-03  TIME:11:12:01+08:00  BARCODE:2102350JAS6TH7001494  RESET 
Num:1                                                                           
--    Reason:Board register, BarCode is 2102350JAS6TH7001494.                   
--    BootMode:NORMAL                                                           
--    BootCode:0x060100ff                                                       
-- 2. DATE:2020-10-03  TIME:11:09:26+08:00  BARCODE:2102350JAS6TH7001494  RESET 
Num:1                                                                           
--    Reason:Board cold reset(COLD Reset)                                       
--    BootMode:NORMAL                                                           
--    BootCode:0x80000101                                                       
----------------------------------------------------------  


如果是CE128整机复位，则主要查看主控板的复位原因，整机复位场景接口板复位原因都为丢心跳所以没有查看的意义。


2、	如果复位原因非上述的几种情况，客户需要快速知悉原因，最小化采集如下命令行提供给研发判断，非紧急场景可以直接跳到步骤3采集。
快速查询---重启问题常见采集命令如下：
V2R2C50之前版本：
用户视图
display start
诊断视图
display device board reset all
display deadloop 20
display exception 20  
display current
display system internal sysmgr  "cat var/log/stackKeyEvent.log" slot X    
display system internal sysmgr  "cat var/log/CE_sdk_info.log" slot X  ---X是复位的槽位
display system internal sysmgr  "cat var/log/lastword.log" slot X    
display system internal sysmgr  "cat var/log/dmesg.log.1.gz" slot X


V2R2C50及之后版本
用户视图
display start
诊断视图
display device board reset all
display deadloop 20
display exception 20  
display current
display system internal sysmgr  "cat var/log/stackKeyEvent.log" slot X 
display system internal sysmgr  "cat var/log/cpdt_sdk_info.log" slot X  ---X是复位的槽位
display system internal sysmgr  "cat var/log/lastword.log" slot X    
display system internal sysmgr  "cat var/log/dmesg.log.1.gz" slot X

3 如果上述命令行研发仍然无法分析出复位原因，则需要采集全量日志，采集内容如下：
d)	logfile目录下复位时间段的log.log 和diag.log以及压缩包日志
e)	display diag回显
f)	诊断视图下敲coll dia info在根目录下生成的diagnostic_information.zip文件，把文件FTP取出来

四、堆叠场景问题
堆叠场景常见的问题为堆叠无法组建、堆叠异常分裂和堆叠快速升级异常处理。
一、堆叠无法组建：
1、	堆叠无法组建，首先排查两侧配置是否正确，主要为DOMAIN、堆叠端口是否正确。通过display stack configuration all来查看。display stack  troubleshooting  也能显示常见的配置错误。
2、	离线配置造成堆叠无法组建，display stack configuration all其中member中带*号的为离线配置，主备两侧的离线配置也需要相同，如果离线配置不同，可以通过clear inactive-configuration all来清除离线配置（清除前建议文本保存所有离线配置）

<SwitchA> display stack configuration 
Oper : Operation 
Conf : Configuration 
* : Offline configuration 
Isolated Port: The port is in stack mode, but does not belong to any Stack-Port 
Attribute Configuration: 
------------------------------------------------------------------- 
MemberID    Domain        Priority         Mode           Enable 
Oper(Conf)   Oper(Conf)  Oper(Conf)  Oper(Conf) Oper 
------------------------------------------------------------------- 
1(1)                10(10)            100(150)      MB(MB)      Disable 
*2(2)               10(10)            100(150)      MB(MB)      Disable   
------------------------------------------------------------------- 
Stack-Port Configuration: 
-------------------------------------------------------------------------------- 
Stack-Port Member Ports
 -------------------------------------------------------------------------------- 
Stack-Port1 10GE1/0/1 10GE1/0/2 10GE2/0/1 10GE2/0/
                                                                         
	
3、	排除配置错误问题后，查看堆叠端口是否能正常UP，在堆叠组建过程中堆叠端口是需要提前UP的，如果现场观察到堆叠端口灯灭，建议更换下端口或者模块线缆尝试修复。
4、	上述排查都未解决问题，采集堆叠两侧的日志提高给研发分析，采集内容如下：
a)	display diag回显
b)	诊断视图下敲coll dia info在根目录下生成的diagnostic_information.zip文件，把文件FTP取出来

二、堆叠异常分裂：
1、	通过display stack  troubleshooting确认基本的分裂原因，确认为堆叠分裂的端口DOWN还是端口DOWN造成的分裂
2、	查看设备异常重启原因（见第三章内容），如果为如下打印说明为堆叠协议通信异常造成，需要采集后台日志分析。

===============================================================================
                 display device board reset 1
===============================================================================

-- 3. DATE:2020-11-03  TIME:10:56:19+02:00  BARCODE:2102350AQB10FC000019  RESET Num:9
--    Reason:The communication between the master and local devices is faulty.(CPU Reset) 
--    BootMode:NORMAL
--    BootCode:0x80000025
                                               
		a)	logfile目录下复位时间段的log.log 和diag.log以及压缩包日志
b)	display diag回显
c)	诊断视图下敲coll dia info在根目录下生成的diagnostic_information.zip文件，把文件FTP取出来

三、堆叠快速升级问题：
1、	display stack  upgrade  status 查看快速升级状态，快速升级存在默认超时时间，CE128为120分钟，TOR为60分钟。对于CE128来说单板完全注册才算结束，如果出现单板长时间不注册并且临近超时，建议直接拔出此单板已免回滚。
2、	如果是快速升级命令报错，可以通过回显确认原因，如果为资源模式不匹配，存在如下可能：
a)	配置了IPV6长掩码模式，设备未重启
b)	配置了CE128 TCAM单板模式，设备未重启
c)	配置了CE128 单板资源模式（大路由，大ARP等），设备未重启
display fei frame resource 和 display system forwarding resource service 能确认具体资源模式。如果存在上诉场景，建议先重启设备或者单板再快速升级。
快速查询---堆叠问题常见采集信息如下：
	display stack configuration all
	display stack troubleshooting  
	display device board reset all
	display stack topo
	display stack 
display stack upgrade status
诊断视图
display stack statistics
display system internal sysmgr  "cat var/log/stackKeyEvent.log" slot X    
display system internal sysmgr  "cat var/log/stack.log" slot X

五、M-LAG场景问题
	M-LAG当前在现网使用中，大部分为转发异常场景，M-LAG无法建立情况较少见，本章节主要阐述在转发异常时主要排查的几个方面。

一、M-LAG两侧配置排查点：
	a、M-LAG和服务器对接，服务器为主备网卡接入，则M-LAG两侧同样为单独端口接入，不需要配置M-LAG。服务器为BOND4负载接入，两侧接入端口必须配置M-LAG。

	b、M-LAG 两种放环模式，根桥模式和V-STP模式，标准配置如下：
      根桥模式：
          PEERLINK口STP DISABLE，两侧配置同样的STP BPDU-MAC，都设置为根桥
	  V-STP模式
		  PEERLINK口STP不关闭，全局配置stp v-stp enable，配置同样的STP BPDU-MAC。
	 通过display stp v-stp 查看PEERLINK状态为Forwarding为正常状态
<HUAWEI> display stp v-stp
Bridge Information: 
   V-STP Mode          :True
   Bridge Mac          :Config=00e0-fc11-1200 / Active=00e0-fc12-1234
   Peer-link Name      :Eth-Trunk1

CIST Global Information: 
   Priority            :Config=0 / Active=0
   Peer-link State     :Forwarding
   CIST Root Times     :Hello 2s MaxAge 20s FwDly 15s MaxHop 20
   CIST Root/ERPC      :0.00e0-fc12-1234 / 0
   CIST RegRoot/IRPC   :0.00e0-fc12-1234 / 0
   Designated Bridge/Port   :0.00e0-fc12-1234 / 0.0
   CIST RootPortId     :0.0(M-lag: Invalid / MAC: 00e0-fc12-1234)
   Virtual Port State  :Active
   Packet Sent         :128
   Packet Received     :128
   TC(Sent/Received)    :1 / 0


	c、M-LAG端口接入错误，两侧M-LAG端口未接到同一台服务器设备，此项可以通过display eth查看对端MAC不一致来确认。

二、M-LAG场景转发故障排查点
	1、M-LAG侧网关PING不通服务器为可能的正常现象，m-lag场景下对服务器来说是负载连到M-LAG两侧，所以存在从M-LAG1 PING服务器时，回包到M-LAG2侧报文被丢弃，该场景下PING不通可以忽略，通过服务器PING网关来确认是否可达。
 
3、	M-LAG 场景下网关MAC地址必须相同，可以通过VRRP配置形成双主或者VLANIF下配置虚MAC方式，如果不同会造成报文流量时断时续，原因为服务器上网关ARP不断变化造成。
4、	M-LAG两侧的MAC和ARP需要同步，如果出现两侧MAC和ARP不一致，可以通过display fei m-lag synchronization packet statistics查看MAC和ARP同步消息情况

<HUAWEI> display fei m-lag synchronization packet statistics
Last Send Success Type    : ARP                                                                        
Last Receive Success Type : MAC                                                                        
Last Send Fail Type       : -                                                                          
Last Receive Fail Type    : -                                                                          
Last Send Fail Reason     : -                                                                          
Last Receive Fail Reason  : -                                                                          
Last Send Success Time    : 2016-08-01 14:08:33                                                        
Last Receive Success Time : 2016-08-01 14:09:36                                                        
Last Send Fail Time       : 0000-00-00 00:00:00                                                        
Last Receive Fail Time    : 0000-00-00 00:00:00                                                        
--------------------------------------------------------------------------------                       
Slot    Type        Send Success    Send Fail    Receive Success    Receive Fail                       
--------------------------------------------------------------------------------                       
2       MAC         0               0            306                0                                  
2       ARP         11              0            0                  0                                  
2       ND          0               0            0                  0                                  
2       STP         0               0            0                  0                                  
2       MC          0               0            0                  0                                  
2       VXFARP      0               0            0                  0                                  
2       VXFVXARP    0               0            0                  0                                  
2       VRRP        0               0            0                  0       		
--------------------------------------------------------------------------------   

5、	转发层面上述排查都未发现异常，建议做流量统计进行定界，M-LAG两侧都需要部署同样的流量统计方式。

快速查询--M-LAG 问题需要采集信息如下
display dfs-group 1 m-lag
display m-lag troubleshooting
display dfs packet statistics 
display fei m-lag synchronization packet statistics
display dfs black-box adjacency 
display dfs black-box error
display dfs black-box high-availability
display dfs black-box fes ifm
display dfs black-box fes m-lag
display dfs black-box fes socket 
display dfs black-box fes trill
display dfs black-box fes v-stp
display fei m-lag log error
display fei m-lag log member
display fei m-lag log peer-link
display fei m-lag log status
display fei m-lag log v-stp
display fei m-lag log vfe

六、ACL场景问题
	ACL在交换机上使用时，主要为ACL不足和ACL不生效。
一、ACL使用不足
	ACL使用不足表现为下发MQC配置时报错资源不足，首先介绍下资源情况：
12800/6870资源情况：
ACL 资源种类包含TCAM、KB、CE等，有任何一种资源不够，ACL业务就会下发失败。
TCAM：TCAM用于存放用户下发的各个ACL规则内容；
CE（Copy Engine）：CE用于从报文头和转发信息提取ACL查找所需的各个字段； 
KB（Key Buffer）：CE提取的字段存储到KB，KB用于存放查找TCAM时构建的查找KEY；
具体规格如下：
ACL资源 	规格
Copy Engine（ CE） 	64个/报文类型
Key Buffer（ KB） 	7个/报文类型
说明
E系列单板的KB资源的规格为7个/KCP；F系
列单板的KB资源的规格为8个/KCP。
Bank 	12个/TCAM
Rule 	12*1024*2*80Bit个/TCAM
说明
E系列单板的Rule资源的规格为12*1024*2*80Bit个/
TCAM；F系列单板的Rule资源的规格为
12*2048*2*80Bit个/TCAM。

TOR上主要为SLICE资源（和BANK相同），各个形态的资源不同，如下规格：
 

	当下发失败时，通过display system  tcam  fail-record slot x先确认具体哪个MQC策略下发失败和失败原因。

<HUAWEI> system-view
[~HUAWEI] display system tcam fail-record slot 1
-----------------------------------------------------------------------------------                                                 
Slot  Chip Time                Service                  ErrInfo                                                                     
-----------------------------------------------------------------------------------                                                 
1     1    2016-03-24 06:40:11 Traffic Policy VLAN      Group resource full                                                         
-----------------------------------------------------------------------------------                                                 
Total: 1     


	当下发失败时，通过display system  tcam  fail-record slot x先确认具体哪个MQC策略下发失败和失败原因。然后通过如下命令采集确认资源剩余情况
	
针对ACL使用不足场景，常见的处理方法有如下几种
A、删减ACL业务或者找不使用ACL的业务方案
通过display system tcam service brief 查看ACL业务，并分析是否可以删除；
如果存在MQC镜像和MQC优先级映射，考虑是否可以用端口镜像和qos镜像替代。

B、使用qos group优化
在多个物理接口、VLAN或者VLANIF下应用相同的traffic policy可以使用qos group进行优化，优化的思路是把同类型接口添加到一个qos group，然后流策略在qos group视图下应用，最终下发的ACL条数只有原来一个接口下数量。

C、调整匹配内容使MQC选相同分组
两个MQC匹配的ACL规则的种类一样，选到的分组必然一样，节约了分组不同占用的slice。

D、使用自定义模板，模板介绍比较复杂，建议联系研发或查看手册中关于ACL自定义模板的介绍。

E、VXLAN 场景 128通过命令行来减少资源
Vxlan场景需要开启两个扩展命令行来节省ACL资源； 
assign forward nvo3  service extend enable
优化业务名称：Ping Packet Pass（优化到cpcar L3分组） ，EVN Packet（优化到cpcar L3分组）
assign forward nvo3 acl extend enable 重启生效，优化业务名称： CPCAR Dci Extend （99）直接删除

快速查询---ACL使用不足需要返回研发需要采集的信息如下：
----- 12800/6870 -----
系统视图：
Display current
Display start
display system tcam bank resource（V2R2C50及以后版本）
display system tcam resource acl
display system tcam acl group resource
诊断视图
display system tcam service brief
display system tcam fail-record
display system tcam service cpcar slot X
V2R2C50及以前版本
fediag slot X chip X "get acl res bank"(V2R2C50及以前版本)
fediag slot X chip X "get acl group info X"(V2R2C50及以前版本)
fediag slot X chip X "get acl group res"(V2R2C50及以前版本)
fediag slot X chip X "get acl res common"(V2R2C50及以前版本)
V2R2C50之后版本
display forward info slot X chip X "get acl res bank "(V2R3版本及以后版本)
display forward info slot X chip X "get acl group info X"(V2R3版本及以后版本)
display forward info slot X chip X "get acl group res"(V2R3版本及以后版本)
display forward info slot X chip X "get acl res common"(V2R3版本及以后版本)

------TOR-------
备注：TOR资源评估命令行汇总：
系统视图：
Display current
Display start
display system tcam bank resource
display system tcam resource acl
诊断视图
display system tcam service brief
display system tcam fail-record
display system tcam service cpcar slot X

V2R2C50及以前版本
fediag slot X chip X "get acl res tcam"(V2R2C50及以前版本)
fediag slot X chip X "get acl group info X"(V2R2C50及以前版本)
fediag slot X chip X "get acl group res"(V2R2C50及以前版本)
V2R2C50之后版本
display forward info slot X chip X "get acl res tcam"(V2R3版本)
display forward info slot X chip X "get acl group info X"(V2R3版本)
display forward info slot X chip X "get acl group res"(V2R3版本)

二、ACL策略不生效
	场景问题为部署镜像或者REMARK动作未生效，一般分为配置问题和软件问题，配置问题为芯片上存在多个策略，造成策略未生效。查看方式如下：
	
[~CE6851HI_147.65-diagnose]display traffic-policy apply-information             
slot 1:                                                                         
------------------------------------------------------------------------------- 
Chip Policy Type/Name         Apply Parameter         GroupId     Priority      
                                                               Group / Service  
------------------------------------------------------------------------------- 
   0 traffic-statistics(IPv4)   10GE1/0/1(In)               37     17 / 102      
   0 traffic-redirect(IPv4)   10GE1/0/1(In)                37     17 / 102      
-------------------------------------------------------------------------------                                                    

	如果两个策略在优先级的Priority分组相同，则只有一个动作可以生效，如果分在不同的组，则可以同时生效。同一流策略应用在不同应用对象时，按照逻辑接口（子接口、VLANIF接口、VBDIF接口）、物理接口、业务实例（VLAN、VPN实例）、全局的优先顺序选择生效。

	其次为MQC的几个场景限制注意点：
•	应用对象为VLANIF接口时：
o	在VLANIF接口应用的流策略，仅对三层单播报文生效。
o	如果流策略中有匹配IPv4字段的规则，则整个流策略只对IPv4单播报文生效；如果流策略中有匹配IPv6字段的规则，则整个流策略只对IPv6单播报文生效；同一个流策略中不支持同时配置匹配IPv4字段和IPv6字段的规则。
o	如果流策略中只有匹配if-match any的规则，则会同时对IPv4和IPv6单播报文生效。
•	针对12800/6870 ,芯片缺省情况下，包含匹配IPv4/IPv6规则的流策略仅对三层转发的IP报文生效。通过本命令开启应用流策略时采用增强模式（traffic-policy ipv4-enhance-mode）的功能后，包含匹配IPv4/IPv6规则的流策略可以同时对二层转发和三层转发的IP报文生效。
快速查询---ACL不生效需要采集的信息如下：
------CE12800/6870---------
V2R2C50及以前版本
display system tcam service cpcar slot X  --- 先获取策略动作的组IP
fediag slot X chip X "get acl group info X"(V2R2C50及以前版本)   --- 获取对应ENTRY索引
fediag slot X chip X "get acl entry info X"(V2R2C50及以前版本)    ---获取ENTRY信息
fediag slot X chip X "get acl entry hit <entryid>"(V2R2C50及以前版本)  --- 查看是否命中
fediag slot X chip X "get acl hit"(V2R2C50及以前版本)  --- 查看全部ACL命中规则
V2R2C50之后版本
display forward info slot X chip X "get acl group info X"  --- 获取对应ENTRY索引
display forward info slot X chip X "get acl entry info X"  ---获取ENTRY信息
display forward info slot X chip X "get acl entry hit <entryid>" --- 查看是否命中
display forward info slot X chip X "get acl hit"  --- 查看全部ACL命中规则


七、二层环路场景问题
	二层场景问题的表现主要为设备环路和二层大量TC报文造成频繁清mac，排查定界方法如下;
	一、环路问题排查过程：
1、	display mac-address flapping 查看漂移的源MAC和漂移端口以及次数：-
[~CE6851HI_147.65-diagnose]display  mac-address flapping                        
MAC Address Flapping Configurations :                                           
------------------------------------------------------------------------------- 
  Flapping detection          : Enable                                          
  Aging  time(s)              : 300                                             
  Quit-VLAN Recover time(m)   : --                                              
  Exclude VLAN-list           : --                                              
  Security level              : Middle                                          
  Exclude BD-list             : --                                              
------------------------------------------------------------------------------- 
S: start time    E: end time    (D): error down                                 
------------------------------------------------------------------------------- 
Time         : S:2020-11-05 23:01:19           E:2020-11-05 23:09:01            
VLAN/BD      : 1/-                                                              
MAC Address  : b443-26d4-467e                                                   
Original-Port: 10GE1/0/19                                                       
Move-Ports   : 10GE1/0/2                                                        
               10GE1/0/21                                                       
MoveNum      : 37281                                                            
------------------------------------------------------------------------------- 
Total items on slot 1: 1   

 
	MoveNum最大值为65535，如果已经达到最大值时将不再增加，如果个数较小可以排除环路可能。在故障场景下可以关闭其中一个成环口做快速恢复。对于环路的MAC查看是否为正常业务MAC，如果为0000-5e00-xxxx等地址，则可能是组网中VRRP地址的漂移，为正常现象。
2、	查看设备STP状态，查看端口流量是否正常
display stp brief  
display interface  brief  | include up
3、	如果设备STP计算状态异常（指定口计算为discard），可以尝试打开DEBUG进行分析
debugging  stp packet all
debugging  stp event
	
	二、清MAC问题过程：
1、	Display stp global查看最近一次STP TC报文的来源


<CE6851HI_147.65>display stp global                                             
 
Protocol Status            :Enabled                                                                                                 
Bpdu-filter default        :Disabled                                                                                                
Tc-protection              :Enabled                                                                                                 
Tc-protection threshold    :1                                                                                                       
Tc-protection interval     :2s                                                                                                      
Edged port default         :Disabled                                                                                                
Pathcost-standard          :Dot1T                                                                                                   
Timer-factor               :3                                                                                                       
Transmit-limit             :6                                                                                                       
Bridge-diameter            :7                                                                                                       
CIST Global Information:                                                                                                            
  Mode                :MSTP                                                                                                         
  CIST Bridge         :32768.0019-7459-3301                                                                                         
  Config Times        :Hello 2s MaxAge 20s FwDly 15s MaxHop 20                                                                      
  Active Times        :Hello 2s MaxAge 20s FwDly 15s MaxHop 20                                                                      
  CIST Root/ERPC      :32768.0019-7459-3301 / 0 (This bridge is the root)                                                           
  CIST RegRoot/IRPC   :32768.0019-7459-3301 / 0 (This bridge is the root)                                                           
  CIST RootPortId     :0.0                                                                                                          
  BPDU-Protection     :Disabled                                                                                                     
  TC or TCN received  :9                                                                                                            
  TC count per hello  :0                                                                                                            
  STP Converge Mode   :Normal                                                                                                       
  Share region-configuration :Enabled                                                                                               
  Time since last TC  :0 days 1h:37m:17s                                                                                            
  Number of TC        :10                                                                                                           
  Last TC occurred    :10GE4/0/12                                                                                                   
  Topo Change Flag    :0



2、display stp tc-bpdu statistics查看端口收到的TC报文情况，注意TC报文计数是跟随端口DOWN清0。

确认到TC报文来源后到相连设备上再做排查，一般情况下TC报文是由于端口闪断造成，大二层网络中一个端口闪断可能会引起整网TC震荡。

快速查询---二层问题需要采集的信息如下：
displau stp global
display stp topology-change 
display stp tc-bpdu statistics 
display stp v-stp  
display fei fe port mapping
cd logfile 
more diag.log
诊断视图
display stp history 
display fei vlan slot X local collect
display fei feisw vlan slot X local collect
display fei mac slot X log mac
display fei mac slot X log macVfeCfg
display fei mac slot X log macVfeTbl
display fei mac slot X log macPath
display fei mac slot X log macVfe
display fei mac slot X log macArp
display fei mac slot X log evn
display fei mac slot X log dual
display fei mac slot X log dualVfe
	

八、三层转发场景问题
	三层问题定位首先介绍下三层表项之间的关系：

 

前缀为FIB表，IIDG为前缀下一跳关联表，ECMP为多下一跳索引表，下一跳nexthop指向出端口。

常见三层问题不通原因：
一、二层协议阻塞，ETH出端口连线错误
这两种是比较容易忽视的场景，建议三层问题定位是都先进行此项判断防止花费大量时间
Display stp brief      --- 查看端口STP状态
Display eth
Display lldp neighbor brief   --- 两条命令配合查询端口连线

二、VM上缺少回程路由。
此项一般通过流量统计确认，常见流量统计方法：

#
acl number 3333
 rule 10 permit icmp source 1.1.1.1 0 destination 1.1.1.2 0 
#
traffic classifier c1 type or
 if-match acl 3333
#
traffic behavior b1
 statistics enable
#
traffic policy p1
 classifier c1 behavior b1 precedence 5 
#
interface 10GE1/0/2
traffic-policy p1 inbound                                                                                                 
 

TOR/128 FD单板支持出方向的流量统计，128 E单板出方向流量统计会造成环回（端口流量大情况严禁配置）。在SDN场景下，设备支持VXLAN报文入方向的内层报文匹配。
#
acl number 3333
 rule 10 permit icmp source 1.1.1.1 0 destination 1.1.1.2 0 
#
traffic classifier c1 type or
 if-match vxlan acl 3333
#
traffic behavior b1
 statistics enable
#
traffic policy p1
 classifier c1 behavior b1 precedence 5 
#
interface 10GE1/0/2
traffic-policy p1 inbound  

三、出口转发异常
a)	display ip routing-table查询路由表确认协议面出口情况


<CE12800_19.4>display  ip routing-table 10.125.3.8 30                                                                        
Proto: Protocol        Pre: Preference                                          
Route Flags: R - relay, D - download to fib, T - to vpn-instance, B - black hole
 route                                                                          
------------------------------------------------------------------------------  
Routing Table : _public_                                                        
Summary Count : 4                                                               
                                                                                
Destination/Mask    Proto   Pre  Cost        Flags NextHop         Interface    
                                                                                
     10.125.3.8/30  EBGP    255  0             RD  100.125.230.18  40GE2/0/35.20
5                                                                               
                    EBGP    255  0             RD  100.125.230.22  40GE2/0/35.20
6                                                                               
                    EBGP    255  0             RD  100.125.232.54  40GE2/0/35.34
1                                                                               
                    EBGP    255  0             RD  100.125.232.58  40GE2/0/35.34


b)	display fei ipv4 route-entry slot x dst-ip x.x.x.x查询到IIDG索引值

<CE12800_19.4>display fei ipv4 route-entry slot 2 dst-ip 10.125.3.8 30                                                          
RE Table:                                                                       
Total number: 1                                                                 
--------------------------------------------------------------------------------
 DestAddr : 10.125.3.8       MaskLen    : 30         FVrfIndex: 0               
 IIDGFlag : 1                Location   : 8          VrIndex  : 0               
 VrfIndex : 0                AttributeID: 0          PathFlag : 1048722         
 IIDGIndex: 2049             ARPNhpIndex: 4294967295                                                

c)	display fei ipv4 iidg slot x iidgindex 2049 查询下一跳索引值，由于为ECMP下一跳为ECMP索引

<CE12800_19.4>display fei ipv4 iidg slot 2 iidgindex 2049                                                                         
IIDG Table:                                                                     
Total number: 1                                                                 
--------------------------------------------------------------------------------
IIDGIndex    VrIndex      VrfIndex     NhpIndex     AttributeID  NhpType   EFlag
     2049          0             0            8               0        1       0

d)	display fei ipv4 ecmp slot 2 ecmpindex 8 查询到4个下一跳索引

<CE12800_19.4>display fei ipv4 ecmp slot 2 ecmpindex 8                          
2020-11-10 20:18:21.265 +16:46                                                  
ECMP Table:                                                                     
Total number: 1                                                                 
--------------------------------------------------------------------------------
 ECMPIndex:  8          VrIndex:  0          RefCnt:  2                         
 Flag     :  2          NhpNum :  4          ResNum:  4                         
 NhpIndex :                                                                     
 0x00000444 0x00000445 0x00000446 0x00000447 0x00000000 0x00000000 0x00000000   
 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000 0x00000000   


e)	display fei ipv4 nexthop slot 2 nhpindex 1092 查询其中一个索引对应的端口情况

<CE12800_19.4>display fei ipv4 nexthop slot 2 nhpindex 1092                     
2020-11-10 20:20:10.132 +16:46                                                  
NHP Table:                                                                      
Total number: 1                                                                 
--------------------------------------------------------------------------------
 NhpIndex  : 1092       VrIndex    : 0          L3IntfIndex: 1218               
 NhpType   : 1          IfType     : 3          TargetBlade: 10                 
 TargetPort: 53         TrunkID    : 0          NhpIP      : 100.125.230.18     
 OutIfIndex: 8997       VcLabel    : 0          RVrfIndex  : 0                  
 EepIndex  : 0          XcIndex    : 0          XcType     : 0                  
 NhlfeID   : 0          NhpVrfIndex: 0          NhpRefCnt  : 2


		上述流程可以确认设备三层转发流程软表是否正常，如果存在异常则说明转发流程有误。

快速查询---三层问题需要采集的信息如下：
	  display fes table 7 slot <slotid> lcfes
display fes table 8
display fes table 9
display fes table 10
display fes table 42
display fes table 43
display fes table 36
display fes table 37
display fes table 111
display fes table 35
display fes table 38
display fes table 541
display fes table 542
display fes table 543
display fes table 544
display fes table 1
display fes table 51
display ip routing-table
display arp
display fei l3 statistics feisw
display fei l3 statistics slot 1
display fei l3 slot 1 log nhp-trace
display fei l3 slot 1 log error
display fei l3 slot 1 log warning
display fei l3 slot 1 log intf-trace
display fei l3 slot 1 log intf-error
display fei l3 slot 1 log arp-trace
display fei l3 slot 1 log arp-error
display fei l3 slot 1 log rm-trace
display fei l3 slot 1 log rm-error
display fei l3 slot 1 log ecmp-trace
display fei l3 slot 1 log ecmp-error
display fei frame record l3-drive slot 1 component fei
display fei frame record l3-event slot 1 component fei
display fei frame record l3-path slot 1 component fei
display fei frame cst FEI_CST_FE_FEC all slot 1 component fei
display fei frame cst FEI_CST_FE_EEDB all slot 1 component fei
display fei frame cst FEI_CST_SOFT_ARP_REF_NHP all slot 1 component fei
display fei l3 interface slot 1
display fei ipv4 arp slot 1
display fei ipv4 route-entry slot 1
display fei ipv4 iidg slot 1
display fei ipv4 nexthop slot 1
display fei ipv4 fe route-entry slot 1
display ip fib slot 1 all-vpn-instance
display fei frame cst FEI_CST_REF_L3INTF_NHP all slot 1 component fei
display fei frame cst FEI_CST_SOFT_ARP_REF_EXTNHP all slot 1 component fei
display fei frame cst FEI_CST_SOFT_ARP_VPNACROSS all slot 1 component fei
display fei frame cst FEI_CST_ID_ARP_FAKE all slot 1 component fei
display fei frame cst FEI_CST_FE_L3INTF all slot 1 component fei
display fei frame cst FEI_CST_RES_EGRL3INTF all slot 1 component fei
display fei frame cst FEI_CST_L3_TBTP2LAG all slot 1 component fei
display fei frame cst FEI_CST_ID_ARP_FAKE all slot 1 component fei
display fei frame record ecmp slot 1 component feisw
display fei frame record fec slot 1 component feisw
display fei frame record encapsulation-database slot 1 component feisw
display fei frame record ecmp slot 1 component fei
display fei frame record fec slot 1 component fei
display fei frame record encapsulation-database slot 1 component fei
九、VXLAN转发场景问题
	VXLAN转发场景主要介绍常见的约束和快速转发定界方法

一、VXLAN转发常见约束
	--- CE12800 VXLAN场景常见约束----------
1、CE12800 EA单板不建议作为承载VXLAN报文。
	2、若承载VXLAN业务的单板为CE-L24XS-EC、CE-L48XS-EC、CE-L24LQ-EC、CE-L48XT-EC、CE-L24LQ-EC1、CE-L08CC-EC、CE-L02LQ-EC、CE-L06LQ-EC时，仅支持通过查询ARP主机表进行VXLAN隧道封装，不支持通过查询最长匹配路由表进行VXLAN隧道封装。
	3、对于CE12800，在单板互通模式为非增强模式时，如果VXLAN流量跨板转发，有可能出现VXLAN流量不通问题。所以单板互通模式为非增强模式时必须配置assign forward nvo3 f-linecard compatibility enable命令才能正常使用VXLAN功能。
	
	-- CE TOR VXLAN场景常见约束--------------
	1、CE6855HI、CE7855EI作为VXLAN三层网关时，只能使用VBDIF接口接入VXLAN网络，否则会导致VXLAN报文无法正常转发
	2、除CE6855HI、CE6870EI、CE6880EI、CE7855EI外，在配置VXLAN三层网关时，需要配置业务环回口。
	3、对于CE6855HI和CE7855EI，在VXLAN多活网关或双活接入场景下：如果VBDIF超过384个后会进行模糊匹配（即只要接收到的IP报文的目的MAC地址与设备上任一VBDIF接口配置的MAC地址相匹配，报文就会进行路由转发
	4、除CE6870EI和CE6880EI外，当允许携带VXLAN报文目的UDP端口号（缺省值为4789）的普通IP报文接入VXLAN网络时，需要在接入端口上执行port nvo3 mode access命令。
二、转发面快速查询方法
三层转发流程查看（CE6855），一键式查看结果
	找到报文的入端口对应的TP（后面要用到的参数）
[~CE6855_leaf_1-diagnose]
display fei fe port mapping 10GE 1/0/48
 
	在该端口根据ACL抓取需要转发的报文
<CE6855_leaf_1>
capture-packet interface 10GE 1/0/5 acl 3333 destination terminal packet-num 1

 

	根据抓到的报文+端口TP，构造一键式定位命令行
[~CE6855_leaf_1-diagnose]
fediag slot 1 chip 0 "get pp info YY addXXXXXXXXX"   
                                               ------YY就是TP、XXXX就是报文

例如：最后一条不加add，就可以显示查询的结果
fediag slot 1 chip 0 "get pp info 5 add48fd8ed5c42100141523470881000014" 
fediag slot 1 chip 0 "get pp info 5 add08004500005408cd0000fe0193bc1401" 
fediag slot 1 chip 0 "get pp info 5 add01140a01010a0000f912062000cd0000" 
fediag slot 1 chip 0 "get pp info 5 00000000000000000000000000000000"  

显示结果：报文信息+入端口信息
 

显示结果：转发信息+下一跳信息
 

---------------------------------------------------------------------------------------------------------------------------------
三层转发流程查看（CE6851），一键式查看结果
	找到报文的入端口对应的TP（后面要用到的参数）
[~CE6851_leaf_2-diagnose]
display fei fe port mapping 10GE 1/0/6
 
	在该端口根据ACL抓取需要转发的报文
<CE6851_leaf_2>
capture-packet interface 10GE 1/0/6 acl 3333 destination terminal packet-num 1
 
	根据抓到的报文+端口TP，构造一键式定位命令行
[~CE6851_leaf_2-diagnose]
fediag slot 1 chip 0 "get pp info YY addXXXXXXXXX"   
                                               ------YY就是TP、XXXX就是报文

例如：最后一条不加add，就可以显示查询的结果
[~CE6851_leaf_2-diagnose]
fediag slot 1 chip 0 "get pp info 6 addc81fbe6fb7013400a30b800c8100000a"
fediag slot 1 chip 0 "get pp info 6 add0800450000540b8d0000ff018ffc0a01"
fediag slot 1 chip 0 "get pp info 6 010a140101140800ee520620038d0000"

显示结果：报文信息+入端口信息
 
显示结果：转发信息+下一跳信息
 



三、VXLAN隧道故障查询方法
	总结起来，隧道建立有几个重要的因素：
1、	EVPN BGP是否建立成功。
2、	两个需要建立隧道的设备之间是否有三类EVPN路由的传递。
EVPN路由之间的收发，可以概括成如下的图，只要有一种路由能互通，隧道就能建立：
 
		常见的隧道侧无法UP原因：
1|、路由不可达；
2、BGP ROUTE-ID重复；
3、BGP下路由引人接收RT错误或者polilcy配置错误：
快速查询---VXLAN问题需要采集的信息如下：
	--------- 隧道相关----------------
display fei tunnel slot X log drv
display fei tunnel slot X log drvtrace
display fei tunnel slot X log error
display fei tunnel slot X log msg
display fei tunnel slot X log path
display fei tunnel slot X log port
display fei tunnel slot X log tblm
display fei tunnel slot X log timer
display fei tunnel slot X log trace
display fei tunnel slot X log vxlanstat
display fei tunnel slot X log vxlanstaterr
display fei tunnel slot X log vxlanstatdrv
display fei tunnel slot X local cmtb
display fei tunnel slot X local data
display fei tunnel slot X local dip
display fei tunnel slot X local eedb
display fei tunnel slot X local extmap
display fei tunnel slot X local f2t
display fei tunnel slot X local g2l
display fei tunnel slot X local oai
display fei tunnel slot X local port
display fei tunnel slot X local res
display fei tunnel slot X local stat
display fei tunnel slot X local tunnel
display fei tunnel slot X local t2f
display fei tunnel slot X local vxlanstate2t
display fei tunnel slot X local vxlanstatt2e
display fes table 1968
display fes table 1390
display fei frame cst FEISW_CST_TNLTRACE all slot 1 component feisw
display fei frame cst FEISW_CST_NVO3_TUNNEL all slot 1 component feisw
display fei frame cst FEI_CST_ID_NVO3_TUNNEL_CFG all slot 1 component fei


----------BDIF相关---------------
display fei bdif slot 4  log arp
display fei bdif slot 4  log drv
display fei bdif slot 4  log drverror
display fei bdif slot 4  log drvtrace
display fei bdif slot 4  log error
display fei bdif slot 4  log path
display fei bdif slot 4  log tblm
display fei bdif slot 4  log trace
display fei bdif slot 4 local bdifipv6en
display fei bdif slot 4 local data
display fei bdif slot 4 local l3gateway
display fei bdif slot 4 local vfevbdif
display fei bdif slot 4 local vfevni2bdif
display fei bdif slot 4 tblm bdif
display fei bdif slot 4 tblm bdifvni
display fei bdif slot 4 tblm distrigw
display fei bdif slot 4 tblm id
display fei bdif slot 4 tblm varp
display fei bdif slot 4 tblm vbdif
display fes table 2059
display fes table 2069
display fes table 2420
display fes table 2405
display fei frame cst FEISW_CST_BDIF_INFO all slot 1 component feisw
display fei frame cst FEISW_CST_BDIF all slot 1 component feisw
display fei frame cst FEI_CST_SOFT_L3INTF_VBDIF all slot 1 component fei
display fei frame cst FEI_CST_SOFT_BDIF_VNI all slot 1 component fei
display fei frame cst FEI_CST_EVPN_VMAC all slot 1 component fei
display fei frame cst FEI_CST_VXLAN_INNER_HASH all slot 1 component fei

转发相关同三层转发查询信息。
