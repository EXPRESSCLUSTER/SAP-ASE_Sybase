# SAP ASE (Sybase ASE) Cluster

## Overview
This article shows how to setup SAP ASE (known as Sybase ASE) cluster.

## Reference
- EXPRESSCLUSTER X for Linux System Requirements  
	https://www.nec.com/en/global/prod/expresscluster/en/overview/sysrep_lx.html?
- EXPRESSCLUSTER X installation  
	https://www.nec.com/en/global/prod/expresscluster/en/support/manuals.html
- SAP ASE installation  
	https://help.sap.com/viewer/bcb0c0752eac4d269c85dc694050df13/16.0.2.0/en-US
- Java  
  https://www.java.com/en/download/

## Cluster System Configuration
2 nodes cluster for Linux with Mirror Disk configuration
```bat
<Public LAN>
 |
 | <Private LAN>
 |  |
 |  |  +--------------------------------+
 +-----| Primary Server                 |
 |  |  |  OS: Linux                     |
 |  +--|  Sybase ASE: SAP ASE 16 SP1    |
 |  |  |  EXPRESSCLUSTER X: 4.1/4.0/3.3 |
 |  |  +--------------------------------+
 |  |
 |  |  +--------------------------------+
 +-----| Secondary Server               |
 |  |  |  OS: Linux                     |
 |  +--|  Sybase ASE: SAP ASE 16 SP1    |
 |  |  |  EXPRESSCLUSTER X: 4.1/4.0/3.3 |
 |  |  +--------------------------------+
 |  |
 |  |  +--------------------------------+
 |  +--| Client machine                 |
 |     |  OS: Windows                   |
 |     |  SW: IE, JRE                   |
 |     +--------------------------------+
 |
[Gateway]
 :
```
### Requirement
- All Primary Server, Secondary Server and Client machine sould be reachable with IP address.
- Ports which EXPRESSCLUSTER requires should be opend
- 2 partitions are required for Mirror Disk Data Partition and Cluster Partition.
	- Data Partition: Depends on Sybase ASE data size
	- Cluster Partition: 1 GB
- Windows Client machine is required to use WebManager (GUI).
- IE and Java version should meet WebManager requirement.

### Sample configuration
- Primary/Secondary Server
	- OS: CentOS 7.4
	- ECX: 3.3.5-1
	- CPU: 2
	- Memory: 8MB
	- Disk
		- /dev/sda: 30GB
		- /dev/sdb:  6GB (depending on SAP ASE database size)

- IP address  

|NW |Primary Server |Secondary Server |fip |Client machine |Gateway |
|--------------|--------------|--------------|--------------|--------------|--------------|
|Public LAN |10.1.1.11 |10.1.1.12 |10.1.1.20 |- |10.1.1.1 |
|Interconnect |192.168.1.11 |192.168.1.12 |- |192.168.1.13 |- |

- Mirror Disk partitions and mount point

|Partition|Primary Server|Secondary Server|Size|
|--------------|--------------|--------------|--------------|
|Cluster Partition|/dev/sdb1|/dev/sdb1|1GB|
|Data Partition|/dev/sdb2|/dev/sdb2|5GB (depending on SAP ASE database size)|
|Mount Point|/mnt/md|/mnt/md|-|

## System setup
### 1. Preparation
#### On both Primary and Secondary Servers
1. Create Cluster Partition and Data Partition  
	```bat
	# fdisk /dev/sdb
	```
1. Create Mount point for Mirror Disk
	```bat
	# mkdir /mnt/md1
	```
1. Crate group and user for SAP ASE and set password for it
	```bat
	# groupadd sap
	# useradd -g sap sap
	# psswd *****
	enter password for sap user
	```
1. Create SAP ASE install directory and change their owener
	```bat
	# mkdir /opt/sap
	# chown sap:sap /opt/sap
	# mkdir /home/sap
	# chown sap:sap /home/sap
	```
1. Change Mount point owner
	```bat
	# chown sap:sap /mnt/md
	```
1. Change share memory size for SAP ASE
	```bat
	# /sbin/sysctl -w kernel.shmmax=268435456
	```  
	\* This size depends on SAP ASE page size. Please refer "SAP ASE installation" in Reference.
	If share memory size is not enough, installation will be failed.
1. Edit hosts file
	```bat
	# vi /etc/hosts
	<virtual hostname (e.g. "asehost")>	10.1.1.20
	```

### 2. Install Java on Clinet machine
#### On Client machine
1. Install Java

### 3. Install EXPRESSCLUSTER X
#### On both Primary and Secondary Server
1. Install EXPRESSCLUSTER X
1. Register Licenses
1. Reboot the server

### 4. Setup basic cluster
#### On Client machine
1. Connect to WebManager
1. Create a new cluster configuration with cluster generation wizard
	- Set cluster name
	- Add Secondary Server to the cluster
	- Add Interconnects
		- Priority: 1
			- Type: Kernel Mode
			- mdc1
			- Primary Server: 192.168.1.11
			- Secondary Server: 192.168.1.12
		- Priority: 2
			- Type: Kernel Mode
			- mdc2
			- Primary Server: 10.1.1.11
			- Secondary Server: 10.1.1.12
	- Add NP resolution resource
		- Type: Ping
		- Ping Target: 10.1.1.1
		- Primary Server: Use
		- Secondary Server: Use
	- Add failover group
	- Add resources to the falover group
		- fip
			- Info
				- Type: floating ip resource
				- Name: fip
			- Dependency: default
			- Recovery Operation: default
			- Details:
				- IP Address: 10.1.1.20
		- md
			- Info
				- Type: mirror disk resource
				- Name: md
			- Dependency: default
			- Recovery Operation: default
			- Datails Partition
				- Mount Point: /mnt/md
				- Data Partition: /dev/sdb2
				- Cluster Partition: /dev/sdb1
				- File System: as you like but should be compatible Sybase ASE
				- Mirror Disk Connect: mdc1, mdc2
1. Apply the Configuration

#### On both Primary and Secondary Server
1. Reboot the server

#### On Client machine
1. Restart IE and connect to WebManager
1. On Operation Mode, start failover group (group) on Primary Server

#### On both Primary and Secondary Server
1. Change Mount point owner
	```bat
	# chown sap:sap /mnt/md
	```

### 5. Install Sybase ASE
#### On Primary Servers
1. Move SAP ASE installer and change its owner
	```bat
	# mv /root/ASE_Suite.linuxamd64.tgz /home/sap
	# chown sap:sap /home/sap/ASE_Suite.linuxamd64.tgz
	```
1. Change user and decompres SAP ASE installer
	```bat
	# su sap
	$ cd /home/sap
	$ tar xvfz ASE_Suite.linuxamd64.tgz
	```
1. Run installer
	```bat
	$ ./setup.bin
	```
1. Set parameters
	- Choose Install Folder
		- Installation Directory: /opt/sap
	- Choose Install Set
		- Typical: Select
	- Configure New Server
		- Configure new SAP ASE: Check
		- Configure new Backup Server: Check
	- Configure Servers with Different User Account
		- No
	- User Configuration Data Directory
		- Data Directory: /opt/sap
	- Configure New SAP ASE (1)
		- SAP ASE Name: <ase name (e.g. "asecluster")>
		- System Administrator's Password: <password for sa user>
		- Host Name: asehost
	- Configure New SAP ASE (2)
		- Master Device:	/mnt/md1/data/master.dat
		- System Procedure Device:	/mnt/md1/data/sysprocs.dat
		- System Device:	/mnt/md1/data/sybsysdb.dat
1. Set environment variable
	```bat
	$ source /opt/sap/SYBASE.sh
	```
1. Check dataserver and backupserver are started
	```bat
	$ /opt/sap/ASE-16_0/install/showserver
	F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
	0 S sap      24843 24842  0  80   0 -  8301 poll_s 08:36 ?        00:00:00 /opt/sap/ASE-16_0/bin/backupserver -e/opt/sap/ASE-16_0/install/testase_BS.log -N25 -C20 -I/opt/sap/interfaces -M/opt/sap/ASE-16_0/bin/sybmultbuf -Stestase_BS -h/opt/sap/ASE-16_0/install/testase_BS.hosts.allow
	0 S sap      21711 21710  1  80   0 - 193279 futex_ 08:34 ?       00:01:03 /opt/sap/ASE-16_0/bin/dataserver -stestase -d/mnt/md1/data/master.dat -e/opt/sap/ASE-16_0/install/testase.log -c/opt/sap/ASE-16_0/testase.cfg -M/opt/sap/ASE-16_0 -N/opt/sap/ASE-16_0/sysam/testase.properties -i/opt/sap
	```
1. Connect to SAP ASE
	```bat
	$ isql -Usa -P<password for sa user> -Sasecluster
	1> select @@version
	2> go
	```
1. Shutdown SAP ASE
	```bat
	1> shutdown
	2> go
	```
#### On Client machine
1. On WebManager Operation Mode, move the group to Secondary Server
#### On Seconday Servers
1. Follow steps ★ on Primary Server
1. Create test table
	```bat
	$ isql -Usa -P<sa password> -Sasecluster
	1> create table testtable (
	2> name varchar(20),
	3> number int)
	4> go
	1> insert into testtable values(test, 1)
	2> go
	1> insert into testtable values ("test",1)
	2> go
	1> select * from testtable
	2> go
	```
1. Shutdown SAP ASE
	```bat
	1> shutdown
	2> go
	```
#### On Client machine
1. On WebManager Operation Mode, move group to Primary Server
#### On Primary Servers
1. Start SAP ASE
	```bat
	$ startserver -f $SYBASE/$SYBASE_ASE/install/RUN_asecluster
	```
1. Confirm the test table is replicated from Secondary Server
	```bat
	$ isql -Usa -P<password for sa user> -Sasecluster
	1> select * from testtable
	2> go
	```
### 6. Prepare to setup SAP ASE cluster
#### On both Primary and Secondary Servers
1. Create key file to encrypt password for sa user which is used by cluster
	```bat
	# su root
	# ssh-keygen
	Generating public/private rsa key pair.
	Enter file in which to save the key (/root/.ssh/id_rsa):
	Enter passphrase (empty for no passphrase): (Enter with empty)
	Enter same passphrase again: (Enter with empty)
	```
1. Create encrypted password file
	```bat
	# echo <sa password> | openssl rsautl -encrypt -inkey ~/.ssh/id_rsa > password.rsa
	```
1. Confirm the password file is created
	```bat
	# openssl rsautl -decrypt -inkey ~/.ssh/id_rsa -in password.rsa
	```
1. Create a sql script to stop SAP ASE
	```bat
	# su sap
	$ mkdir /home/sap/scripts
	$ vi /home/sap/scripts/
	shutdown
	go
	```
### 7. Setup SAP ASE cluster
#### On Client machine
1. On WebManager Condiguration Mode, move group to Secondary Server
1. Right click group under "Groups" in the left tree and select Add Resource
1. Add exec resource
	- Info
		- Type: EXEC resuorce
		- Name: exec_sapase
	- Dependency: default
	- Recovery Operation: default
	- Details
		- Start Script: ★
		- Stop Script: ★
		- Tuning
			- Parameter
				- Start Script
					- Synchronous: Check
					- Timeout: 180
				- Stot Script
					- Synchronous: Check
					- Timeout: 180
			- Maintenance
				- Log Output Path: /opt/nec/clusterpro/log/exec_sapase.log
				- Rotate Log: Check
				- Rotate Size: 1000000 byte
1. Apply the Configuration
1. On Operation Mode, start exec_sapase

### 8. Scripts
#### Start Script
```bat
#! /bin/sh
#***************************************
#*              start.sh               *
#***************************************

#ulimit -s unlimited

echo "Start"

#Start SAP ASE dataserver
source /opt/sap/SYBASE.sh
su - sap -c "source /opt/sap/SYBASE.sh; startserver -f $SYBASE/$SYBASE_ASE/install/RUN_testase"

#Check dataserver is started
sleep 10
result=`su - sap -c "source /opt/sap/SYBASE.sh; showserver"`
echo $result | grep dataserver
if [ $? -eq 1 ]
then
    echo "ERROR: SAP ASE failed to start."
    echo "Finish"
    exit 1
fi
echo "SAP ASE started."
echo "Finish"
exit 0
```

#### Start Script
```bat
#! /bin/sh
#***************************************
#*               stop.sh               *
#***************************************

#ulimit -s unlimited

echo "Start"

PASSWORD=$(openssl rsautl -decrypt -inkey ~/.ssh/id_rsa -in password.rsa)
su - sap -c "source /opt/sap/SYBASE.sh; isql -Usa -P$PASSWORD -Stestase -i /home/sap/shutdownase"
sleep 10
result=`su - sap -c "source /opt/sap/SYBASE.sh; showserver"`
echo $result | grep dataserver
if [ $? -eq 0 ]
then
    echo "SAP ASE failed to stop"
    echo "Finish"
    exit 1
fi

echo "SAP ASE stopped"
echo "Finish"
exit 0
```

## Note
