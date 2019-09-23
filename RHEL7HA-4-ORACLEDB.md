# Red Hat Enterprise Linux 7.x High Availability 实现 Oracle Database 单实例的高可用

## 思路 
1. 由 Pacemaker 实现 VIP 、卷/文件系统、数据库实例和数据库监听器的故障恢复
2. 用 ansible 实现自动化地安装和配置
3. 现实中的一些问题与解决方案

## 准备步骤
共 3 台机器:
* 192.168.56.1  rhel7            NTP服务器、DNS服务器和测试客户端 
* 192.168.56.71 rhel7ha1.sa.rh   HA服务器1, RHEL 7.x minimal 安装 
* 192.168.56.72 rhel7ha2.sa.rh   HA服务器2, RHEL 7.x minimal 安装 
共享磁盘 挂载到 2 个 HA 节点 
网络连通, 192.168.56.70 用于 vip 


1. 在 192.168.56.1 上配置 chronyd 服务做为时钟源
```shell
# yum install chrony
# cat << EOF > /etc/chrony.conf
>server UPSTREAM_TIME_SOURCE iburst
>driftfile /var/lib/chrony/drift
>makestep 1.0 3
>rtcsync
>allow 192.168.56.0/24
>keyfile /etc/chrony.keys
>logdir /var/log/chrony
>stratumweight 0
>bindcmdaddress 127.0.0.1
>bindcmdaddress ::1
>commandkey 1
>generatecommandkey
>EOF

# firewall-cmd --permanent --add-service=ntp
# firewall-cmd --reload
# chronyc keygen >> /etc/chrony.keys
# systemctl start chronyd
# systemctl enable chronyd
```

2. 在 2 个 HA 节点上拷贝 chrony.keys 文件并配置 ntp 服务
```shell
# yum instll chrony
# cat << EOF > /etc/chrony.conf
>server 192.168.56.1 iburst
>driftfile /var/lib/chrony/drift
>makestep 1.0 3
>rtcsync
>keyfile /etc/chrony.keys
>logdir /var/log/chrony
>EOF

# echo CHRONY_KEY >> /etc/chrony.keys
# systemctl start chronyd
# systemctl enable chronyd
# chronyc sources
# chronyc tracking
```
** 注, CHRONY_KEY 是上一步所生成并保存在 /etc/chrony.keys 中的 key **

3. 在 2 个 HA 节点上建 /u01 目录, 建 oinstall 和 dba 组, 建 oracle 用户
```shell
# mkdir -p /u01
# groupadd oinstall
# groupadd dba
# useradd oracle -g oinstall -G dba
# echo ORACLE_PASSWORD | passwd --stdin oracle
# chown oracle:oinstall /u01
```
** 注, ORACLE_PASSWORD 是为 oracle 用户设置的密码 **


## 思路1的步骤: 
1. 在 2 个 HA 节点上设置服务器名称和命名解析。若DNS服务器可用, 可忽略此步骤
```shell
# cat << EOF >> /etc/hosts
> 192.168.56.71 rhel7ha1.sa.rh rhel7ha1
> 192.168.56.72 rhel7ha2.sa.rh rhel7ha2
> 192.168.56.70 rhel7vip.sa.rh rhel7vip
> EOF
```

2. 在 2 个 HA 节点上安装 HA 所需要的包并启动服务
```shell
# yum install pcs pacemaker fence-agents-all
# systemctl start pcsd
# systemctl enable pcsd
```

3. 在 2 个 HA 节点上打开防火墙的对应服务端口
```shell
# firewall-cmd --permanent --add-service=high-availability
# firewall-cmd --permanent --add-port=7410/udp
# firewall-cmd --reload
# firewall-cmd --list-all
```
** 防火墙配合 HA 增加了 high-availability 服务, udp 7410 端口给 stonith kdump 设备使用 **

4. 在 2 个 HA 节点上设置hacluster用户的密码
```shell
# echo HACLUSTER_PASSWORD | passwd --stdin hacluster
```
** 注, HACLUSTER_PASSWORD 是为 hacluster 用户设置的密码 **


5. 在 HA1 节点上注册两个 HA 节点的集群服务并创建集群
```shell
# pcs cluster auth rhel7ha1.sa.rh rhel7ha2.sa.rh
Username: hacluster
Password:
rhel7ha2.sa.rh: Authorized
rhel7ha1.sa.rh: Authorized
# 
# pcs cluster setup --start --name cluster01 rhel7ha1.sa.rh rhel7ha2.sa.rh
Destroying cluster on nodes: rhel7ha1.sa.rh, rhel7ha2.sa.rh...
rhel7ha1.sa.rh: Stopping Cluster (pacemaker)...
rhel7ha2.sa.rh: Stopping Cluster (pacemaker)...
rhel7ha2.sa.rh: Successfully destroyed cluster
rhel7ha1.sa.rh: Successfully destroyed cluster

Sending 'pacemaker_remote authkey' to 'rhel7ha1.sa.rh', 'rhel7ha2.sa.rh'
rhel7ha1.sa.rh: successful distribution of the file 'pacemaker_remote authkey'
rhel7ha2.sa.rh: successful distribution of the file 'pacemaker_remote authkey'
Sending cluster config files to the nodes...
rhel7ha1.sa.rh: Succeeded
rhel7ha2.sa.rh: Succeeded

Starting cluster on nodes: rhel7ha1.sa.rh, rhel7ha2.sa.rh...
rhel7ha1.sa.rh: Starting Cluster (corosync)...
rhel7ha2.sa.rh: Starting Cluster (corosync)...
rhel7ha1.sa.rh: Starting Cluster (pacemaker)...
rhel7ha2.sa.rh: Starting Cluster (pacemaker)...

Synchronizing pcsd certificates on nodes rhel7ha1.sa.rh, rhel7ha2.sa.rh...
rhel7ha2.sa.rh: Success
rhel7ha1.sa.rh: Success
Restarting pcsd on the nodes in order to reload the certificates...
rhel7ha2.sa.rh: Success
rhel7ha1.sa.rh: Success
```

6. 在 HA2 节点上验证集群状态
```shell
# pcs status
Cluster name: cluster01

WARNINGS:
No stonith devices and stonith-enabled is not false

Stack: corosync
Current DC: rhel7ha2.sa.rh (version 1.1.19-8.el7-c3c624ea3d) - partition with quorum
Last updated: Sun Sep  8 16:47:45 2019
Last change: Sun Sep  8 16:47:24 2019 by hacluster via crmd on rhel7ha2.sa.rh

2 nodes configured
0 resources configured

Online: [ rhel7ha1.sa.rh rhel7ha2.sa.rh ]

No resources


Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
# 
# pcs cluster status
Cluster Status:
 Stack: corosync
 Current DC: rhel7ha2.sa.rh (version 1.1.19-8.el7-c3c624ea3d) - partition with quorum
 Last updated: Sun Sep  8 16:48:05 2019
 Last change: Sun Sep  8 16:47:24 2019 by hacluster via crmd on rhel7ha2.sa.rh
 2 nodes configured
 0 resources configured

PCSD Status:
  rhel7ha2.sa.rh: Online
  rhel7ha1.sa.rh: Online
```

7. 在 HA1 节点上, 设置 stonith kdump 设备
```shell
# pcs stonith create kdump fence_kdump pcmk_reboot_action="off" pcmk_host_list="rhel7ha1.sa.rh rhel7ha2.sa.rh"
```

** 设置 stonith kdump 可以防止节点故障时, 集群软件直接停机, 从而不产生 kdump 文件的问题. **
或者通过以下命令禁用 stonith **
```shell
# pcs property set stonith-enabled=false
```

8. 在 HA1 节点上, 增加 vip 资源
```shell
# pcs resource create vip ocf:heartbeat:IPaddr2 ip=192.168.56.70 cidr_netmask=24 nic=enp0s3 op monitor interval=10s --group=oracle
# 
# pcs status
Cluster name: cluster01
Stack: corosync
Current DC: rhel7ha2.sa.rh (version 1.1.19-8.el7-c3c624ea3d) - partition with quorum
Last updated: Sun Sep  8 17:19:01 2019
Last change: Sun Sep  8 17:18:42 2019 by root via cibadmin on rhel7ha1.sa.rh

2 nodes configured
2 resources configured

Online: [ rhel7ha1.sa.rh rhel7ha2.sa.rh ]

Full list of resources:

 kdump	(stonith:fence_kdump):	Started rhel7ha1.sa.rh
 Resource Group: oracle
     vip	(ocf::heartbeat:IPaddr2):	Started rhel7ha2.sa.rh

Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
```
9. 在 2 个 HA 节点上设置 udev 规则, 指定共享盘的别名为 /dev/sharedisk01
```shell
# lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda             8:0    0   20G  0 disk 
├─sda1          8:1    0  200M  0 part /boot/efi
├─sda2          8:2    0  800M  0 part /boot
└─sda3          8:3    0   19G  0 part 
  ├─rhel-root 253:0    0   17G  0 lvm  /
  └─rhel-swap 253:1    0    2G  0 lvm  [SWAP]
sdb             8:16   0    8G  0 disk 
sr0            11:0    1 1024M  0 rom
```

** 看到共享盘目前是 /dev/sdb , 为了避免添加其它盘时共享盘名字可能会变化, 设置 udev 规则, 指定共享盘的别名为 /dev/sharedisk01 **
```shell
# udevadm info -q path -n /dev/sdb
/devices/pci0000:00/0000:00:14.0/host4/target4:0:0/4:0:0:0/block/sdb
# 
# udevadm info --query=all -p /devices/pci0000:00/0000:00:14.0/host4/target4:0:0/4:0:0:0/block/sdb
P: /devices/pci0000:00/0000:00:14.0/host4/target4:0:0/4:0:0:0/block/sdb
N: sdb
S: disk/by-path/pci-0000:00:14.0-scsi-0:0:0:0
E: DEVLINKS=/dev/disk/by-path/pci-0000:00:14.0-scsi-0:0:0:0
E: DEVNAME=/dev/sdb
E: DEVPATH=/devices/pci0000:00/0000:00:14.0/host4/target4:0:0/4:0:0:0/block/sdb
E: DEVTYPE=disk
E: ID_BUS=scsi
E: ID_MODEL=HARDDISK
E: ID_MODEL_ENC=HARDDISK\x20\x20\x20\x20\x20\x20\x20\x20
E: ID_PATH=pci-0000:00:14.0-scsi-0:0:0:0
E: ID_PATH_TAG=pci-0000_00_14_0-scsi-0_0_0_0
E: ID_REVISION=1.0
E: ID_SCSI=1
E: ID_TYPE=disk
E: ID_VENDOR=VBOX
E: ID_VENDOR_ENC=VBOX\x20\x20\x20\x20
E: MAJOR=8
E: MINOR=16
E: MPATH_SBIN_PATH=/sbin
E: SUBSYSTEM=block
E: TAGS=:systemd:
E: USEC_INITIALIZED=76425

# cat << EOF > /etc/udev/rules.d/99-shared-disk-rule.rules
> ACTION=="add|change", KERNEL=="sd?", ENV{ID_PATH_TAG}=="pci-0000_00_14_0-scsi-0_0_0_0", SYMLINK+="sharedisk01"
> EOF
```

10. 在 2 个 HA 节点上配置 lvm 启动
```shell
# echo 'volume_list = [ ROOT_VG ]' >> /etc/lvm/lvm.conf
# dracut -H -f /boot/initramfs-$(uname -r).img $(uname -r)
# reboot
```
** 注, ROOT_VG 指 /dev/sda 上的 ROOT 分区的 VG 名字, 在此阻止启动时 lvm 加载共享 VG , 而是在 pacemaker 服务中加载共享 VG **

11. 在 HA2 节点上, 增加共享卷和文件系统资源
```shell
# pvcreate /dev/sharedisk01
# vgcreate shared_vg /dev/sharedisk01
# lvcreate -l 100%FREE -n ha_lv shared_vg
# lvs
# mkfs.xfs /dev/shared_vg/ha_lv
# lvmconf --enable-halvm --services --startstopservices
# pcs resource create svg ocf:heartbeat:LVM volgrpname=shared_vg exclusive=true --group=oracle
# pcs resource create sfs ocf:heartbeat:Filesystem device="/dev/shared_vg/ha_lv" directory="/u01" fstype="xfs" --group=oracle
```

12. 确认资源在 HA2 节点上, 安装 Oracle 数据库到 /u01 目录并在文件系统上创建实例
```shell
# pcs status
Cluster name: cluster01
Stack: corosync
Current DC: rhel7ha2.sa.rh (version 1.1.19-8.el7-c3c624ea3d) - partition with quorum
Last updated: Mon Sep 23 09:44:38 2019
Last change: Sun Sep 22 17:00:17 2019 by root via cibadmin on rhel7ha2.sa.rh

2 nodes configured
4 resources configured

Online: [ rhel7ha1.sa.rh rhel7ha2.sa.rh ]

Full list of resources:

 kdump	(stonith:fence_kdump):	Started rhel7ha1.sa.rh
 Resource Group: oracle
     vip	(ocf::heartbeat:IPaddr2):	Started rhel7ha2.sa.rh
     svg	(ocf::heartbeat:LVM):	Started rhel7ha2.sa.rh
     sfs	(ocf::heartbeat:Filesystem):	Started rhel7ha2.sa.rh

Daemon Status:
  corosync: active/disabled
  pacemaker: active/disabled
  pcsd: active/enabled
```
** 注, 若 pcs status 出现 Error: cluster is not currently running on this node , 通过 pcs cluster start --all 启动 **

安装 Oracle 12c 数据库, SID 为 acme ,  LISTENER 为 listener_acme , ORACLE_HOME 为 /u01/app/oracle/product/12.1.0/dbhome_1
步骤请参考 Deploying Oracle Database 12c on Red Hat Enterprise Linux 7


13. 在 HA2 节点上, 增加 Oracle 数据库资源
```shell
# pcs resource create ora ocf:heartbeat:oracle sid="acme" --group=oracle
# pcs resource update ora monpassword="MON_PASSWORD" monuser="c##ocfmon" monprofile="default"
# pcs resource create lsnr_ora ocf:heartbeat:oralsnr sid="acme" listener="listener_acme" --group=oracle
```
** 注, MON_PASSWORD 是 Oracle 12c container 数据库的用户 ocfmon 的密码 **


14. 在 HA2 节点上，测试切换资源
```shell
# pcs resource move oracle rhel7ha1.sa.rh
# pcs status
```


## 附录
1. 查看 HA 支持的资源
```shell
# pcs resource list ocf:heartbeat
ocf:heartbeat:aliyun-vpc-move-ip - Move IP within a VPC of the Aliyun ECS
ocf:heartbeat:apache - Manages an Apache Web server instance
ocf:heartbeat:aws-vpc-move-ip - Move IP within a VPC of the AWS EC2
ocf:heartbeat:awseip - Amazon AWS Elastic IP Address Resource Agent
ocf:heartbeat:awsvip - Amazon AWS Secondary Private IP Address Resource Agent
ocf:heartbeat:azure-lb - Answers Azure Load Balancer health probe requests
ocf:heartbeat:clvm - clvmd
ocf:heartbeat:conntrackd - This resource agent manages conntrackd
ocf:heartbeat:CTDB - CTDB Resource Agent
ocf:heartbeat:db2 - Resource Agent that manages an IBM DB2 LUW databases in
                    Standard role as primitive or in HADR roles as master/slave
                    configuration. Multiple partitions are supported.
ocf:heartbeat:Delay - Waits for a defined timespan
ocf:heartbeat:dhcpd - Chrooted ISC DHCP server resource agent.
ocf:heartbeat:docker - Docker container resource agent.
ocf:heartbeat:Dummy - Example stateless resource agent
ocf:heartbeat:ethmonitor - Monitors network interfaces
ocf:heartbeat:exportfs - Manages NFS exports
ocf:heartbeat:Filesystem - Manages filesystem mounts
ocf:heartbeat:galera - Manages a galara instance
ocf:heartbeat:garbd - Manages a galera arbitrator instance
ocf:heartbeat:iface-vlan - Manages VLAN network interfaces.
ocf:heartbeat:IPaddr - Manages virtual IPv4 and IPv6 addresses (Linux specific version)
ocf:heartbeat:IPaddr2 - Manages virtual IPv4 and IPv6 addresses (Linux specific version)
ocf:heartbeat:IPsrcaddr - Manages the preferred source address for outgoing IP packets
ocf:heartbeat:iSCSILogicalUnit - Manages iSCSI Logical Units (LUs)
ocf:heartbeat:iSCSITarget - iSCSI target export agent
ocf:heartbeat:LVM - Controls the availability of an LVM Volume Group
ocf:heartbeat:LVM-activate - This agent activates/deactivates logical volumes.
ocf:heartbeat:lvmlockd - This agent manages the lvmlockd daemon
ocf:heartbeat:MailTo - Notifies recipients by email in the event of resource takeover
ocf:heartbeat:mysql - Manages a MySQL database instance
ocf:heartbeat:nagios - Nagios resource agent
ocf:heartbeat:named - Manages a named server
ocf:heartbeat:nfsnotify - sm-notify reboot notifications
ocf:heartbeat:nfsserver - Manages an NFS server
ocf:heartbeat:nginx - Manages an Nginx web/proxy server instance
ocf:heartbeat:NodeUtilization - Node Utilization
ocf:heartbeat:oraasm - Oracle ASM resource agent
ocf:heartbeat:oracle - Manages an Oracle Database instance
ocf:heartbeat:oralsnr - Manages an Oracle TNS listener
ocf:heartbeat:pgsql - Manages a PostgreSQL database instance
ocf:heartbeat:portblock - Block and unblocks access to TCP and UDP ports
ocf:heartbeat:postfix - Manages a highly available Postfix mail server instance
ocf:heartbeat:rabbitmq-cluster - rabbitmq clustered
ocf:heartbeat:redis - Redis server
ocf:heartbeat:Route - Manages network routes
ocf:heartbeat:rsyncd - Manages an rsync daemon
ocf:heartbeat:SendArp - Broadcasts unsolicited ARP announcements
ocf:heartbeat:slapd - Manages a Stand-alone LDAP Daemon (slapd) instance
ocf:heartbeat:Squid - Manages a Squid proxy server instance
ocf:heartbeat:sybaseASE - Sybase ASE Failover Instance
ocf:heartbeat:symlink - Manages a symbolic link
ocf:heartbeat:tomcat - Manages a Tomcat servlet environment instance
ocf:heartbeat:vdo-vol - VDO resource agent
ocf:heartbeat:VirtualDomain - Manages virtual domains through the libvirt virtualization framework
ocf:heartbeat:Xinetd - Manages a service of Xinetd
```

2. 查看 stonith 设备
```shell
# pcs stonith list
fence_amt_ws - Fence agent for AMT (WS)
fence_apc - Fence agent for APC over telnet/ssh
fence_apc_snmp - Fence agent for APC, Tripplite PDU over SNMP
fence_bladecenter - Fence agent for IBM BladeCenter
fence_brocade - Fence agent for HP Brocade over telnet/ssh
fence_cisco_mds - Fence agent for Cisco MDS
fence_cisco_ucs - Fence agent for Cisco UCS
fence_compute - Fence agent for the automatic resurrection of OpenStack compute instances
fence_drac5 - Fence agent for Dell DRAC CMC/5
fence_eaton_snmp - Fence agent for Eaton over SNMP
fence_emerson - Fence agent for Emerson over SNMP
fence_eps - Fence agent for ePowerSwitch
fence_evacuate - Fence agent for the automatic resurrection of OpenStack compute instances
fence_heuristics_ping - Fence agent for ping-heuristic based fencing
fence_hpblade - Fence agent for HP BladeSystem
fence_ibmblade - Fence agent for IBM BladeCenter over SNMP
fence_idrac - Fence agent for IPMI
fence_ifmib - Fence agent for IF MIB
fence_ilo - Fence agent for HP iLO
fence_ilo2 - Fence agent for HP iLO
fence_ilo3 - Fence agent for IPMI
fence_ilo3_ssh - Fence agent for HP iLO over SSH
fence_ilo4 - Fence agent for IPMI
fence_ilo4_ssh - Fence agent for HP iLO over SSH
fence_ilo5 - Fence agent for IPMI
fence_ilo5_ssh - Fence agent for HP iLO over SSH
fence_ilo_moonshot - Fence agent for HP Moonshot iLO
fence_ilo_mp - Fence agent for HP iLO MP
fence_ilo_ssh - Fence agent for HP iLO over SSH
fence_imm - Fence agent for IPMI
fence_intelmodular - Fence agent for Intel Modular
fence_ipdu - Fence agent for iPDU over SNMP
fence_ipmilan - Fence agent for IPMI
fence_kdump - fencing agent for use with kdump crash recovery service
fence_mpath - Fence agent for multipath persistent reservation
fence_rhevm - Fence agent for RHEV-M REST API
fence_rsa - Fence agent for IBM RSA
fence_rsb - I/O Fencing agent for Fujitsu-Siemens RSB
fence_sbd - Fence agent for sbd
fence_scsi - Fence agent for SCSI persistent reservation
fence_virt - Fence agent for virtual machines
fence_vmware_rest - Fence agent for VMware REST API
fence_vmware_soap - Fence agent for VMWare over SOAP API
fence_wti - Fence agent for WTI
fence_xvm - Fence agent for virtual machines
```

## 参考
[Common Users in a CDB](https://docs.oracle.com/database/121/ADMQS/GUID-DA54EBE5-43EF-4B09-B8CC-FAABA335FBB8.htm)
[Use udev rules in RHEL 7](https://access.redhat.com/solutions/1135513)
[Configure Kdump with RHEL 7 HA](https://access.redhat.com/articles/67570)
[Configure fence_kdump in RHEL HA](https://access.redhat.com/solutions/2876971)
[RHEL 7 HA Administrator Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/high_availability_add-on_administration/index#ch-service-HAAA)
[Deploying Oracle Database 12c on Red Hat Enterprise Linux 7](https://access.redhat.com/articles/1282303)
