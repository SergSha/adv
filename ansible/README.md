### 02-Ansible - Установка на Ubuntu и CentOS

### 04-Ansible - Подключение к серверам LINUX

В домашней директории создадим каталог ansible:
...
[user@localhost ~]$ mkdir ./ansible
[user@localhost ~]$
...

Переходим в этот каталог:
...
[user@localhost ~]$ mkdir ./ansible
[user@localhost ansible]$
...

Создадим Vagrantfile:
...
[user@localhost ansible]$ vi Vagrantfile
...

...
# -*- mode: ruby -*-
# vi: set ft=ruby :

hosts = {
  "host0" => "192.168.1.10",
  "host1" => "192.168.1.11",
  "host2" => "192.168.1.12"
}

Vagrant.configure("2") do |config|
  hosts.each do |name, ip|
    config.vm.define name do |machine|
      machine.vm.box = "centos/7"
      machine.vm.hostname = "%s" % name
      machine.vm.network "public_network", bridge: "wlp2s0", ip: ip
      machine.vm.provider "virtualbox" do |v|
          v.name = name
          v.customize ["modifyvm", :id, "--memory", 256]
      end
      machine.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
            sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
            systemctl restart sshd
          SHELL
    end
  end
end
...

Создадим пару ключей (без парольной фразы) для группы серверов staging_servers:
...
[user@localhost ansible]$ ssh-keygen -t rsa -b 4096 -f /home/user/.ssh/staging
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/user/.ssh/staging.
Your public key has been saved in /home/user/.ssh/staging.pub.
The key fingerprint is:
SHA256:pa/43dWfBwVECKNy7u1zCr/XU3/1Ik8vD1Oq7TfWzmw user@localhost.localdomain
The key's randomart image is:
+---[RSA 4096]----+
|          o. +o  |
|         . .. .  |
|      . o .    . |
|       + o      .|
|        S      ..|
|       . o    .+o|
|        o o  .*o*|
|       . *..+==OE|
|      ..o **o+=X@|
+----[SHA256]-----+
[user@localhost ansible]$
...

Создадим пару ключей (без парольной фразы) для группы серверов prod_servers:
...
[user@localhost ansible]$ ssh-keygen -t rsa -b 4096 -f /home/user/.ssh/product
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/user/.ssh/product.
Your public key has been saved in /home/user/.ssh/product.pub.
The key fingerprint is:
SHA256:mKsYfM+Rao0+UgqqXaW+Tgu2AIdmBAR7nex1FLZcVB8 user@localhost.localdomain
The key's randomart image is:
+---[RSA 4096]----+
|=.      +oo.. E  |
|.. o . + o   . . |
|... + . +     .  |
|.o . . +         |
|oo. . + S        |
|=o  .o o         |
|o.=o=o+          |
|.+oX+*..         |
|o +=Ooo          |
+----[SHA256]-----+
[user@localhost ansible]$
...

### 06-Ansible - Правила создания файла Inventory

Создадим inventory файл:
...
[user@localhost ansible]$ vi ./inventory
...

...
[staging_servers]
host0 ansible_host=192.168.1.10 ansible_port=22 ansible_user=vagrant ansible_private_key_file=/home/user/.ssh/staging

[prod_servers]
host1 ansible_host=192.168.1.11
host2 ansible_host=192.168.1.12

[prod_servers:vars]
ansible_port=22
ansible_user=vagrant
ansible_private_key_file=/home/user/.ssh/product
...

Для образца inventory файл можно сформировать следующим образом:
...
10.50.1.1
10.50.1.2

[staging_DB]
192.168.1.1
192.169.1.2

[staging_WEB]
192.168.2.1
192.168.2.2

[staging_APP]
192.168.3.1
192.168.3.2

[staging_ALL:children]
staging_DB
staging_WEB
staging_APP


[prod_DB]
10.10.1.1
10.10.1.2

[prod_WEB]
10.10.2.1
10.10.2.2

[prod_APP]
10.10.3.1
10.10.3.2

[prod_ALL:children]
prod_DB
prod_WEB
prod_APP


[DB_ALL:children]
staging_DB
prod_DB

[WEB_ALL:children]
staging_WEB
prod_WEB

[APP_ALL:children]
staging_APP
prod_APP

[RAZNOE:children]
APP_ALL
DB_ALL

[RAZNOE:vars]
message=Hello
...

Можно запустить команду для всех серверов:
...
[user@localhost ansible]$ ansible -i ./inventory all -m ping
...

Чтобы не указывать inventory файл и не спрашивал разрешения при первом входе в сервера, создадим ansible конфиг файл:
...
[user@localhost ansible]$ vi ansible.cfg
...

...
[defaults]
host_key_checking = false        # Игнорировать проверку ключа при первом входе
inventory         = ./inventory  # Указываем путь к inventory файлу
...

Запустим модуль ping для всех серверов:
...
[user@localhost ansible]$ ansible all -m ping
host0 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
host1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
host2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
[user@localhost ansible]$
...
Наблюдаем, от всех серверов получаем ответ "pong", то есть все сервера доступны.

Можно запустить команду для определенной группы серверов, например, prod_servers:
...
[user@localhost ansible]$ ansible prod_servers -m ping
host1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
host2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "ping": "pong"
}
[user@localhost ansible]$
...

Можно просмотреть inventory список:
...
[user@localhost ansible]$ ansible-inventory --list
{
    "_meta": {
        "hostvars": {
            "host0": {
                "ansible_host": "192.168.1.10", 
                "ansible_port": 22, 
                "ansible_private_key_file": "/home/user/.ssh/staging", 
                "ansible_user": "vagrant"
            }, 
            "host1": {
                "ansible_host": "192.168.1.11", 
                "ansible_port": 22, 
                "ansible_private_key_file": "/home/user/.ssh/product", 
                "ansible_user": "vagrant"
            }, 
            "host2": {
                "ansible_host": "192.168.1.12", 
                "ansible_port": 22, 
                "ansible_private_key_file": "/home/user/.ssh/product", 
                "ansible_user": "vagrant"
            }
        }
    }, 
    "all": {
        "children": [
            "prod_servers", 
            "staging_servers", 
            "ungrouped"
        ]
    }, 
    "prod_servers": {
        "hosts": [
            "host1", 
            "host2"
        ]
    }, 
    "staging_servers": {
        "hosts": [
            "host0"
        ]
    }
}
[user@localhost ansible]$
...

### 07-Ansible - Запуск Ad-Hoc Комманд

Ad-Hoc команды это разовые команды, которые не в Playbook, с помощью которых можно что-то разово выполнить, посмотреть, настроить. Например, с помощью модуля setup можно посмотреть очень подробные сведения серверов, например,группы staging_servers:
...
[user@localhost ansible]$ ansible staging_servers -m setup
host0 | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "192.168.1.10", 
            "10.0.2.15"
        ], 
        "ansible_all_ipv6_addresses": [
            "fe80::a00:27ff:fee8:2a82", 
            "fe80::5054:ff:fe4d:77d3"
        ], 
        "ansible_apparmor": {
            "status": "disabled"
        }, 
        "ansible_architecture": "x86_64", 
        "ansible_bios_date": "12/01/2006", 
        "ansible_bios_version": "VirtualBox", 
        "ansible_cmdline": {
            "BOOT_IMAGE": "/boot/vmlinuz-3.10.0-1127.el7.x86_64", 
            "LANG": "en_US.UTF-8", 
            "biosdevname": "0", 
            "console": "ttyS0,115200n8", 
            "crashkernel": "auto", 
            "elevator": "noop", 
            "net.ifnames": "0", 
            "no_timer_check": true, 
            "ro": true, 
            "root": "UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15"
        }, 
        "ansible_date_time": {
            "date": "2022-07-30", 
            "day": "30", 
            "epoch": "1659208276", 
            "hour": "19", 
            "iso8601": "2022-07-30T19:11:16Z", 
            "iso8601_basic": "20220730T191116706375", 
            "iso8601_basic_short": "20220730T191116", 
            "iso8601_micro": "2022-07-30T19:11:16.706375Z", 
            "minute": "11", 
            "month": "07", 
            "second": "16", 
            "time": "19:11:16", 
            "tz": "UTC", 
            "tz_offset": "+0000", 
            "weekday": "Saturday", 
            "weekday_number": "6", 
            "weeknumber": "30", 
            "year": "2022"
        }, 
        "ansible_default_ipv4": {
            "address": "10.0.2.15", 
            "alias": "eth0", 
            "broadcast": "10.0.2.255", 
            "gateway": "10.0.2.2", 
            "interface": "eth0", 
            "macaddress": "52:54:00:4d:77:d3", 
            "mtu": 1500, 
            "netmask": "255.255.255.0", 
            "network": "10.0.2.0", 
            "type": "ether"
        }, 
        "ansible_default_ipv6": {}, 
        "ansible_device_links": {
            "ids": {
                "sda": [
                    "ata-VBOX_HARDDISK_VB58d9b4cf-7e6df1ca"
                ], 
                "sda1": [
                    "ata-VBOX_HARDDISK_VB58d9b4cf-7e6df1ca-part1"
                ]
            }, 
            "labels": {}, 
            "masters": {}, 
            "uuids": {
                "sda1": [
                    "1c419d6c-5064-4a2b-953c-05b2c67edb15"
                ]
            }
        }, 
        "ansible_devices": {
            "sda": {
                "holders": [], 
                "host": "IDE interface: Intel Corporation 82371AB/EB/MB PIIX4 IDE (rev 01)", 
                "links": {
                    "ids": [
                        "ata-VBOX_HARDDISK_VB58d9b4cf-7e6df1ca"
                    ], 
                    "labels": [], 
                    "masters": [], 
                    "uuids": []
                }, 
                "model": "VBOX HARDDISK", 
                "partitions": {
                    "sda1": {
                        "holders": [], 
                        "links": {
                            "ids": [
                                "ata-VBOX_HARDDISK_VB58d9b4cf-7e6df1ca-part1"
                            ], 
                            "labels": [], 
                            "masters": [], 
                            "uuids": [
                                "1c419d6c-5064-4a2b-953c-05b2c67edb15"
                            ]
                        }, 
                        "sectors": "83884032", 
                        "sectorsize": 512, 
                        "size": "40.00 GB", 
                        "start": "2048", 
                        "uuid": "1c419d6c-5064-4a2b-953c-05b2c67edb15"
                    }
                }, 
                "removable": "0", 
                "rotational": "1", 
                "sas_address": null, 
                "sas_device_handle": null, 
                "scheduler_mode": "noop", 
                "sectors": "83886080", 
                "sectorsize": "512", 
                "size": "40.00 GB", 
                "support_discard": "0", 
                "vendor": "ATA", 
                "virtual": 1
            }
        }, 
        "ansible_distribution": "CentOS", 
        "ansible_distribution_file_parsed": true, 
        "ansible_distribution_file_path": "/etc/redhat-release", 
        "ansible_distribution_file_variety": "RedHat", 
        "ansible_distribution_major_version": "7", 
        "ansible_distribution_release": "Core", 
        "ansible_distribution_version": "7.8", 
        "ansible_dns": {
            "nameservers": [
                "10.0.2.3"
            ]
        }, 
        "ansible_domain": "", 
        "ansible_effective_group_id": 1000, 
        "ansible_effective_user_id": 1000, 
        "ansible_env": {
            "HOME": "/home/vagrant", 
            "LANG": "C", 
            "LC_ALL": "C", 
            "LC_MEASUREMENT": "ru_RU.UTF-8", 
            "LC_MESSAGES": "C", 
            "LC_MONETARY": "ru_RU.UTF-8", 
            "LC_NUMERIC": "ru_RU.UTF-8", 
            "LC_PAPER": "ru_RU.UTF-8", 
            "LC_TIME": "ru_RU.UTF-8", 
            "LESSOPEN": "||/usr/bin/lesspipe.sh %s", 
            "LOGNAME": "vagrant", 
            "LS_COLORS": "rs=0:di=38;5;27:ln=38;5;51:mh=44;38;5;15:pi=40;38;5;11:so=38;5;13:do=38;5;5:bd=48;5;232;38;5;11:cd=48;5;232;38;5;3:or=48;5;232;38;5;9:mi=05;48;5;232;38;5;15:su=48;5;196;38;5;15:sg=48;5;11;38;5;16:ca=48;5;196;38;5;226:tw=48;5;10;38;5;16:ow=48;5;10;38;5;21:st=48;5;21;38;5;15:ex=38;5;34:*.tar=38;5;9:*.tgz=38;5;9:*.arc=38;5;9:*.arj=38;5;9:*.taz=38;5;9:*.lha=38;5;9:*.lz4=38;5;9:*.lzh=38;5;9:*.lzma=38;5;9:*.tlz=38;5;9:*.txz=38;5;9:*.tzo=38;5;9:*.t7z=38;5;9:*.zip=38;5;9:*.z=38;5;9:*.Z=38;5;9:*.dz=38;5;9:*.gz=38;5;9:*.lrz=38;5;9:*.lz=38;5;9:*.lzo=38;5;9:*.xz=38;5;9:*.bz2=38;5;9:*.bz=38;5;9:*.tbz=38;5;9:*.tbz2=38;5;9:*.tz=38;5;9:*.deb=38;5;9:*.rpm=38;5;9:*.jar=38;5;9:*.war=38;5;9:*.ear=38;5;9:*.sar=38;5;9:*.rar=38;5;9:*.alz=38;5;9:*.ace=38;5;9:*.zoo=38;5;9:*.cpio=38;5;9:*.7z=38;5;9:*.rz=38;5;9:*.cab=38;5;9:*.jpg=38;5;13:*.jpeg=38;5;13:*.gif=38;5;13:*.bmp=38;5;13:*.pbm=38;5;13:*.pgm=38;5;13:*.ppm=38;5;13:*.tga=38;5;13:*.xbm=38;5;13:*.xpm=38;5;13:*.tif=38;5;13:*.tiff=38;5;13:*.png=38;5;13:*.svg=38;5;13:*.svgz=38;5;13:*.mng=38;5;13:*.pcx=38;5;13:*.mov=38;5;13:*.mpg=38;5;13:*.mpeg=38;5;13:*.m2v=38;5;13:*.mkv=38;5;13:*.webm=38;5;13:*.ogm=38;5;13:*.mp4=38;5;13:*.m4v=38;5;13:*.mp4v=38;5;13:*.vob=38;5;13:*.qt=38;5;13:*.nuv=38;5;13:*.wmv=38;5;13:*.asf=38;5;13:*.rm=38;5;13:*.rmvb=38;5;13:*.flc=38;5;13:*.avi=38;5;13:*.fli=38;5;13:*.flv=38;5;13:*.gl=38;5;13:*.dl=38;5;13:*.xcf=38;5;13:*.xwd=38;5;13:*.yuv=38;5;13:*.cgm=38;5;13:*.emf=38;5;13:*.axv=38;5;13:*.anx=38;5;13:*.ogv=38;5;13:*.ogx=38;5;13:*.aac=38;5;45:*.au=38;5;45:*.flac=38;5;45:*.mid=38;5;45:*.midi=38;5;45:*.mka=38;5;45:*.mp3=38;5;45:*.mpc=38;5;45:*.ogg=38;5;45:*.ra=38;5;45:*.wav=38;5;45:*.axa=38;5;45:*.oga=38;5;45:*.spx=38;5;45:*.xspf=38;5;45:", 
            "MAIL": "/var/mail/vagrant", 
            "PATH": "/usr/local/bin:/usr/bin", 
            "PWD": "/home/vagrant", 
            "SELINUX_LEVEL_REQUESTED": "", 
            "SELINUX_ROLE_REQUESTED": "", 
            "SELINUX_USE_CURRENT_RANGE": "", 
            "SHELL": "/bin/bash", 
            "SHLVL": "2", 
            "SSH_CLIENT": "192.168.1.152 43374 22", 
            "SSH_CONNECTION": "192.168.1.152 43374 192.168.1.10 22", 
            "SSH_TTY": "/dev/pts/0", 
            "TERM": "xterm-256color", 
            "USER": "vagrant", 
            "XDG_RUNTIME_DIR": "/run/user/1000", 
            "XDG_SESSION_ID": "12", 
            "XMODIFIERS": "@im=ibus", 
            "_": "/usr/bin/python"
        }, 
        "ansible_eth0": {
            "active": true, 
            "device": "eth0", 
            "features": {
                "busy_poll": "off [fixed]", 
                "fcoe_mtu": "off [fixed]", 
                "generic_receive_offload": "on", 
                "generic_segmentation_offload": "on", 
                "highdma": "off [fixed]", 
                "hw_tc_offload": "off [fixed]", 
                "l2_fwd_offload": "off [fixed]", 
                "large_receive_offload": "off [fixed]", 
                "loopback": "off [fixed]", 
                "netns_local": "off [fixed]", 
                "ntuple_filters": "off [fixed]", 
                "receive_hashing": "off [fixed]", 
                "rx_all": "off", 
                "rx_checksumming": "off", 
                "rx_fcs": "off", 
                "rx_gro_hw": "off [fixed]", 
                "rx_udp_tunnel_port_offload": "off [fixed]", 
                "rx_vlan_filter": "on [fixed]", 
                "rx_vlan_offload": "on", 
                "rx_vlan_stag_filter": "off [fixed]", 
                "rx_vlan_stag_hw_parse": "off [fixed]", 
                "scatter_gather": "on", 
                "tcp_segmentation_offload": "on", 
                "tx_checksum_fcoe_crc": "off [fixed]", 
                "tx_checksum_ip_generic": "on", 
                "tx_checksum_ipv4": "off [fixed]", 
                "tx_checksum_ipv6": "off [fixed]", 
                "tx_checksum_sctp": "off [fixed]", 
                "tx_checksumming": "on", 
                "tx_fcoe_segmentation": "off [fixed]", 
                "tx_gre_csum_segmentation": "off [fixed]", 
                "tx_gre_segmentation": "off [fixed]", 
                "tx_gso_partial": "off [fixed]", 
                "tx_gso_robust": "off [fixed]", 
                "tx_ipip_segmentation": "off [fixed]", 
                "tx_lockless": "off [fixed]", 
                "tx_nocache_copy": "off", 
                "tx_scatter_gather": "on", 
                "tx_scatter_gather_fraglist": "off [fixed]", 
                "tx_sctp_segmentation": "off [fixed]", 
                "tx_sit_segmentation": "off [fixed]", 
                "tx_tcp6_segmentation": "off [fixed]", 
                "tx_tcp_ecn_segmentation": "off [fixed]", 
                "tx_tcp_mangleid_segmentation": "off", 
                "tx_tcp_segmentation": "on", 
                "tx_udp_tnl_csum_segmentation": "off [fixed]", 
                "tx_udp_tnl_segmentation": "off [fixed]", 
                "tx_vlan_offload": "on [fixed]", 
                "tx_vlan_stag_hw_insert": "off [fixed]", 
                "udp_fragmentation_offload": "off [fixed]", 
                "vlan_challenged": "off [fixed]"
            }, 
            "hw_timestamp_filters": [], 
            "ipv4": {
                "address": "10.0.2.15", 
                "broadcast": "10.0.2.255", 
                "netmask": "255.255.255.0", 
                "network": "10.0.2.0"
            }, 
            "ipv6": [
                {
                    "address": "fe80::5054:ff:fe4d:77d3", 
                    "prefix": "64", 
                    "scope": "link"
                }
            ], 
            "macaddress": "52:54:00:4d:77:d3", 
            "module": "e1000", 
            "mtu": 1500, 
            "pciid": "0000:00:03.0", 
            "promisc": false, 
            "speed": 1000, 
            "timestamping": [
                "tx_software", 
                "rx_software", 
                "software"
            ], 
            "type": "ether"
        }, 
        "ansible_eth1": {
            "active": true, 
            "device": "eth1", 
            "features": {
                "busy_poll": "off [fixed]", 
                "fcoe_mtu": "off [fixed]", 
                "generic_receive_offload": "on", 
                "generic_segmentation_offload": "on", 
                "highdma": "off [fixed]", 
                "hw_tc_offload": "off [fixed]", 
                "l2_fwd_offload": "off [fixed]", 
                "large_receive_offload": "off [fixed]", 
                "loopback": "off [fixed]", 
                "netns_local": "off [fixed]", 
                "ntuple_filters": "off [fixed]", 
                "receive_hashing": "off [fixed]", 
                "rx_all": "off", 
                "rx_checksumming": "off", 
                "rx_fcs": "off", 
                "rx_gro_hw": "off [fixed]", 
                "rx_udp_tunnel_port_offload": "off [fixed]", 
                "rx_vlan_filter": "on [fixed]", 
                "rx_vlan_offload": "on", 
                "rx_vlan_stag_filter": "off [fixed]", 
                "rx_vlan_stag_hw_parse": "off [fixed]", 
                "scatter_gather": "on", 
                "tcp_segmentation_offload": "on", 
                "tx_checksum_fcoe_crc": "off [fixed]", 
                "tx_checksum_ip_generic": "on", 
                "tx_checksum_ipv4": "off [fixed]", 
                "tx_checksum_ipv6": "off [fixed]", 
                "tx_checksum_sctp": "off [fixed]", 
                "tx_checksumming": "on", 
                "tx_fcoe_segmentation": "off [fixed]", 
                "tx_gre_csum_segmentation": "off [fixed]", 
                "tx_gre_segmentation": "off [fixed]", 
                "tx_gso_partial": "off [fixed]", 
                "tx_gso_robust": "off [fixed]", 
                "tx_ipip_segmentation": "off [fixed]", 
                "tx_lockless": "off [fixed]", 
                "tx_nocache_copy": "off", 
                "tx_scatter_gather": "on", 
                "tx_scatter_gather_fraglist": "off [fixed]", 
                "tx_sctp_segmentation": "off [fixed]", 
                "tx_sit_segmentation": "off [fixed]", 
                "tx_tcp6_segmentation": "off [fixed]", 
                "tx_tcp_ecn_segmentation": "off [fixed]", 
                "tx_tcp_mangleid_segmentation": "off", 
                "tx_tcp_segmentation": "on", 
                "tx_udp_tnl_csum_segmentation": "off [fixed]", 
                "tx_udp_tnl_segmentation": "off [fixed]", 
                "tx_vlan_offload": "on [fixed]", 
                "tx_vlan_stag_hw_insert": "off [fixed]", 
                "udp_fragmentation_offload": "off [fixed]", 
                "vlan_challenged": "off [fixed]"
            }, 
            "hw_timestamp_filters": [], 
            "ipv4": {
                "address": "192.168.1.10", 
                "broadcast": "192.168.1.255", 
                "netmask": "255.255.255.0", 
                "network": "192.168.1.0"
            }, 
            "ipv6": [
                {
                    "address": "fe80::a00:27ff:fee8:2a82", 
                    "prefix": "64", 
                    "scope": "link"
                }
            ], 
            "macaddress": "08:00:27:e8:2a:82", 
            "module": "e1000", 
            "mtu": 1500, 
            "pciid": "0000:00:08.0", 
            "promisc": false, 
            "speed": 1000, 
            "timestamping": [
                "tx_software", 
                "rx_software", 
                "software"
            ], 
            "type": "ether"
        }, 
        "ansible_fibre_channel_wwn": [], 
        "ansible_fips": false, 
        "ansible_form_factor": "Other", 
        "ansible_fqdn": "host0", 
        "ansible_hostname": "host0", 
        "ansible_hostnqn": "", 
        "ansible_interfaces": [
            "lo", 
            "eth1", 
            "eth0"
        ], 
        "ansible_is_chroot": true, 
        "ansible_iscsi_iqn": "", 
        "ansible_kernel": "3.10.0-1127.el7.x86_64", 
        "ansible_kernel_version": "#1 SMP Tue Mar 31 23:36:51 UTC 2020", 
        "ansible_lo": {
            "active": true, 
            "device": "lo", 
            "features": {
                "busy_poll": "off [fixed]", 
                "fcoe_mtu": "off [fixed]", 
                "generic_receive_offload": "on", 
                "generic_segmentation_offload": "on", 
                "highdma": "on [fixed]", 
                "hw_tc_offload": "off [fixed]", 
                "l2_fwd_offload": "off [fixed]", 
                "large_receive_offload": "off [fixed]", 
                "loopback": "on [fixed]", 
                "netns_local": "on [fixed]", 
                "ntuple_filters": "off [fixed]", 
                "receive_hashing": "off [fixed]", 
                "rx_all": "off [fixed]", 
                "rx_checksumming": "on [fixed]", 
                "rx_fcs": "off [fixed]", 
                "rx_gro_hw": "off [fixed]", 
                "rx_udp_tunnel_port_offload": "off [fixed]", 
                "rx_vlan_filter": "off [fixed]", 
                "rx_vlan_offload": "off [fixed]", 
                "rx_vlan_stag_filter": "off [fixed]", 
                "rx_vlan_stag_hw_parse": "off [fixed]", 
                "scatter_gather": "on", 
                "tcp_segmentation_offload": "on", 
                "tx_checksum_fcoe_crc": "off [fixed]", 
                "tx_checksum_ip_generic": "on [fixed]", 
                "tx_checksum_ipv4": "off [fixed]", 
                "tx_checksum_ipv6": "off [fixed]", 
                "tx_checksum_sctp": "on [fixed]", 
                "tx_checksumming": "on", 
                "tx_fcoe_segmentation": "off [fixed]", 
                "tx_gre_csum_segmentation": "off [fixed]", 
                "tx_gre_segmentation": "off [fixed]", 
                "tx_gso_partial": "off [fixed]", 
                "tx_gso_robust": "off [fixed]", 
                "tx_ipip_segmentation": "off [fixed]", 
                "tx_lockless": "on [fixed]", 
                "tx_nocache_copy": "off [fixed]", 
                "tx_scatter_gather": "on [fixed]", 
                "tx_scatter_gather_fraglist": "on [fixed]", 
                "tx_sctp_segmentation": "on", 
                "tx_sit_segmentation": "off [fixed]", 
                "tx_tcp6_segmentation": "on", 
                "tx_tcp_ecn_segmentation": "on", 
                "tx_tcp_mangleid_segmentation": "on", 
                "tx_tcp_segmentation": "on", 
                "tx_udp_tnl_csum_segmentation": "off [fixed]", 
                "tx_udp_tnl_segmentation": "off [fixed]", 
                "tx_vlan_offload": "off [fixed]", 
                "tx_vlan_stag_hw_insert": "off [fixed]", 
                "udp_fragmentation_offload": "on", 
                "vlan_challenged": "on [fixed]"
            }, 
            "hw_timestamp_filters": [], 
            "ipv4": {
                "address": "127.0.0.1", 
                "broadcast": "", 
                "netmask": "255.0.0.0", 
                "network": "127.0.0.0"
            }, 
            "ipv6": [
                {
                    "address": "::1", 
                    "prefix": "128", 
                    "scope": "host"
                }
            ], 
            "mtu": 65536, 
            "promisc": false, 
            "timestamping": [
                "rx_software", 
                "software"
            ], 
            "type": "loopback"
        }, 
        "ansible_local": {}, 
        "ansible_lsb": {}, 
        "ansible_machine": "x86_64", 
        "ansible_machine_id": "bea7f74edb5759459dc7f9b6744cc2a4", 
        "ansible_memfree_mb": 4, 
        "ansible_memory_mb": {
            "nocache": {
                "free": 119, 
                "used": 116
            }, 
            "real": {
                "free": 4, 
                "total": 235, 
                "used": 231
            }, 
            "swap": {
                "cached": 0, 
                "free": 2047, 
                "total": 2047, 
                "used": 0
            }
        }, 
        "ansible_memtotal_mb": 235, 
        "ansible_mounts": [
            {
                "block_available": 9701755, 
                "block_size": 4096, 
                "block_total": 10480385, 
                "block_used": 778630, 
                "device": "/dev/sda1", 
                "fstype": "xfs", 
                "inode_available": 20943269, 
                "inode_total": 20971008, 
                "inode_used": 27739, 
                "mount": "/", 
                "options": "rw,seclabel,relatime,attr2,inode64,noquota", 
                "size_available": 39738388480, 
                "size_total": 42927656960, 
                "uuid": "1c419d6c-5064-4a2b-953c-05b2c67edb15"
            }
        ], 
        "ansible_nodename": "host0", 
        "ansible_os_family": "RedHat", 
        "ansible_pkg_mgr": "yum", 
        "ansible_proc_cmdline": {
            "BOOT_IMAGE": "/boot/vmlinuz-3.10.0-1127.el7.x86_64", 
            "LANG": "en_US.UTF-8", 
            "biosdevname": "0", 
            "console": [
                "tty0", 
                "ttyS0,115200n8"
            ], 
            "crashkernel": "auto", 
            "elevator": "noop", 
            "net.ifnames": "0", 
            "no_timer_check": true, 
            "ro": true, 
            "root": "UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15"
        }, 
        "ansible_processor": [
            "0", 
            "GenuineIntel", 
            "Intel(R) Core(TM) i5-6200U CPU @ 2.30GHz"
        ], 
        "ansible_processor_cores": 1, 
        "ansible_processor_count": 1, 
        "ansible_processor_threads_per_core": 1, 
        "ansible_processor_vcpus": 1, 
        "ansible_product_name": "VirtualBox", 
        "ansible_product_serial": "NA", 
        "ansible_product_uuid": "NA", 
        "ansible_product_version": "1.2", 
        "ansible_python": {
            "executable": "/usr/bin/python", 
            "has_sslcontext": true, 
            "type": "CPython", 
            "version": {
                "major": 2, 
                "micro": 5, 
                "minor": 7, 
                "releaselevel": "final", 
                "serial": 0
            }, 
            "version_info": [
                2, 
                7, 
                5, 
                "final", 
                0
            ]
        }, 
        "ansible_python_version": "2.7.5", 
        "ansible_real_group_id": 1000, 
        "ansible_real_user_id": 1000, 
        "ansible_selinux": {
            "config_mode": "enforcing", 
            "mode": "enforcing", 
            "policyvers": 31, 
            "status": "enabled", 
            "type": "targeted"
        }, 
        "ansible_selinux_python_present": true, 
        "ansible_service_mgr": "systemd", 
        "ansible_ssh_host_key_ecdsa_public": "AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBB0eu/CZxD1p6O68VpulVvdTM0PsJ92yNKths6OmJI5K6fvWY9fjjZGhXvRmTnPzow4o595fW+Nuo7/tXCxtSX0=", 
        "ansible_ssh_host_key_ed25519_public": "AAAAC3NzaC1lZDI1NTE5AAAAIFjToRwY8gdim/G3jTgF1dVHzLOr5bE9Bq+gXNvQg5b0", 
        "ansible_ssh_host_key_rsa_public": "AAAAB3NzaC1yc2EAAAADAQABAAABAQCmHIdCJk5poG6OdNqrF5RlS/BFAcgYEnFntzB4jfEKUCbDUWPZOzcSaz6iPzivWNvhZTQCpNZi5X0FDCuuBk+rPpFDHqASzR/mclZbIyaKWMVOnSBFBfTIKh7gucSc45PgjUBA1u3By1XtisU5NYplX2T2zPoUt8F3E+cYyEg0flDaq8gA/N8RWSaTirFWt8fMPkR1kdp2t7FawfF0QCbMz4zKF6b5thFPJgRlgvensOydJl8hd8k1Beqizq+Cj8cL/jiZbOaZihpfjswdYrBh0HXG9C6RH7LPnGojNFM94qKU0FaZyoCA+SAlr63GbzZxlRPXSdSR38L3+Iae9PkB", 
        "ansible_swapfree_mb": 2047, 
        "ansible_swaptotal_mb": 2047, 
        "ansible_system": "Linux", 
        "ansible_system_capabilities": [
            ""
        ], 
        "ansible_system_capabilities_enforced": "True", 
        "ansible_system_vendor": "innotek GmbH", 
        "ansible_uptime_seconds": 11405, 
        "ansible_user_dir": "/home/vagrant", 
        "ansible_user_gecos": "vagrant", 
        "ansible_user_gid": 1000, 
        "ansible_user_id": "vagrant", 
        "ansible_user_shell": "/bin/bash", 
        "ansible_user_uid": 1000, 
        "ansible_userspace_architecture": "x86_64", 
        "ansible_userspace_bits": "64", 
        "ansible_virtualization_role": "guest", 
        "ansible_virtualization_type": "virtualbox", 
        "discovered_interpreter_python": "/usr/bin/python", 
        "gather_subset": [
            "all"
        ], 
        "module_setup": true
    }, 
    "changed": false
}
[user@localhost ansible]$
...

Можно запустить модуль shell:
...
[user@localhost ansible]$ ansible all -m shell -a "pwd"
host1 | CHANGED | rc=0 >>
/home/vagrant
host0 | CHANGED | rc=0 >>
/home/vagrant
host2 | CHANGED | rc=0 >>
/home/vagrant
[user@localhost ansible]$
...

...
[user@localhost ansible]$ ansible all -m shell -a "uptime"
host0 | CHANGED | rc=0 >>
 19:15:41 up  3:14,  1 user,  load average: 0.00, 0.01, 0.05
host1 | CHANGED | rc=0 >>
 19:15:41 up  3:13,  1 user,  load average: 0.00, 0.01, 0.04
host2 | CHANGED | rc=0 >>
 19:15:42 up  3:13,  1 user,  load average: 0.00, 0.01, 0.05
[user@localhost ansible]$
...

Вместо модуля shell можно использовать command, но невозможно будет использовать такие знаки как "<", ">", "|".

Для примера копирования на все сервера или группы серверов создадим файл, например, hello.txt:
...
[user@localhost ansible]$ echo Hello > hello.txt
[user@localhost ansible]$
...

Копируем созданный файл на все сервера, где src - откуда копируем, dest - куда копируем, mode - права на файл:
...
[user@localhost ansible]$ ansible all -m copy -a "src=./hello.txt dest=/home mode=777"
host2 | FAILED! => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "msg": "Destination /home not writable"
}
host1 | FAILED! => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "msg": "Destination /home not writable"
}
host0 | FAILED! => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "msg": "Destination /home not writable"
}
[user@localhost ansible]$
...
Из строки "msg": "Destination /home not writable" видим, что скопировать не удалось, так как мы пытались разместить файл в директорию /home, а не в саму директорию пользователя /home/user/. Для этого нам нужны права root, то есть с параметром "-b", от слова "become", то есть sudo:
...
[user@localhost ansible]$ ansible all -m copy -a "src=./hello.txt dest=/home mode=777" -b
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "dest": "/home/hello.txt", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "09f7e02f1290be211da707a266f153b3", 
    "mode": "0777", 
    "owner": "root", 
    "secontext": "unconfined_u:object_r:user_home_dir_t:s0", 
    "size": 6, 
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1659213860.71-22085-178986656583149/source", 
    "state": "file", 
    "uid": 0
}
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "dest": "/home/hello.txt", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "09f7e02f1290be211da707a266f153b3", 
    "mode": "0777", 
    "owner": "root", 
    "secontext": "unconfined_u:object_r:user_home_dir_t:s0", 
    "size": 6, 
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1659213860.66-22083-52330120429969/source", 
    "state": "file", 
    "uid": 0
}
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "dest": "/home/hello.txt", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "09f7e02f1290be211da707a266f153b3", 
    "mode": "0777", 
    "owner": "root", 
    "secontext": "unconfined_u:object_r:user_home_dir_t:s0", 
    "size": 6, 
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1659213860.72-22081-240026082870701/source", 
    "state": "file", 
    "uid": 0
}
[user@localhost ansible]$
...
Теперь из строки "changed": true видим, что файл удалось скопировать во все сервера.

Попробуем повторить команду:
...
[user@localhost ansible]$ ansible all -m copy -a "src=./hello.txt dest=/home mode=777" -b
host2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "dest": "/home/hello.txt", 
    "gid": 0, 
    "group": "root", 
    "mode": "0777", 
    "owner": "root", 
    "path": "/home/hello.txt", 
    "secontext": "unconfined_u:object_r:user_home_dir_t:s0", 
    "size": 6, 
    "state": "file", 
    "uid": 0
}
host0 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "dest": "/home/hello.txt", 
    "gid": 0, 
    "group": "root", 
    "mode": "0777", 
    "owner": "root", 
    "path": "/home/hello.txt", 
    "secontext": "unconfined_u:object_r:user_home_dir_t:s0", 
    "size": 6, 
    "state": "file", 
    "uid": 0
}
host1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "checksum": "1d229271928d3f9e2bb0375bd6ce5db6c6d348d9", 
    "dest": "/home/hello.txt", 
    "gid": 0, 
    "group": "root", 
    "mode": "0777", 
    "owner": "root", 
    "path": "/home/hello.txt", 
    "secontext": "unconfined_u:object_r:user_home_dir_t:s0", 
    "size": 6, 
    "state": "file", 
    "uid": 0
}
[user@localhost ansible]$
...
Как видим из строки "changed": false, изменений не произошло, так как файл уже существует.

Проверим, что файл hello.txt действительно существует в директории /home во всех серверах:
...
[user@localhost ansible]$ ansible all -m shell -a "ls -la /home"
host0 | CHANGED | rc=0 >>
total 4
drwxr-xr-x.  3 root    root     38 Jul 30 20:44 .
dr-xr-xr-x. 18 root    root    255 Jul 30 16:01 ..
-rwxrwxrwx.  1 root    root      6 Jul 30 20:44 hello.txt
drwx------.  4 vagrant vagrant  90 Jul 30 16:10 vagrant
host2 | CHANGED | rc=0 >>
total 4
drwxr-xr-x.  3 root    root     38 Jul 30 20:44 .
dr-xr-xr-x. 18 root    root    255 Jul 30 16:03 ..
-rwxrwxrwx.  1 root    root      6 Jul 30 20:44 hello.txt
drwx------.  4 vagrant vagrant  90 Jul 30 16:10 vagrant
host1 | CHANGED | rc=0 >>
total 4
drwxr-xr-x.  3 root    root     38 Jul 30 20:44 .
dr-xr-xr-x. 18 root    root    255 Jul 30 16:02 ..
-rwxrwxrwx.  1 root    root      6 Jul 30 20:44 hello.txt
drwx------.  4 vagrant vagrant  90 Jul 30 16:10 vagrant
[user@localhost ansible]$
...

Теперь давай удалим файл hello.txt, но с модулем file, где path - путь к файлу, state - статус, то есть absent - отсутствовать (не забываем параметр "-b" для sudo):
...
[user@localhost ansible]$ ansible all -m file -a "path=/home/hello.txt state=absent" -b
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "path": "/home/hello.txt", 
    "state": "absent"
}
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "path": "/home/hello.txt", 
    "state": "absent"
}
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "path": "/home/hello.txt", 
    "state": "absent"
}
[user@localhost ansible]$
...
Судя по строке "changed": true команда по удалению файла прошла успешно.

Давай для примера на всех серверах скачаем что-нибудь с интернета в домашнюю директорию:
...
[user@localhost ansible]$ ansible all -m get_url -a "url=https://mirror.yandex.ru/centos/7.9.2009/isos/x86_64/0_README.txt dest=/home/vagrant"
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum_dest": null, 
    "checksum_src": "87c0d9dc6757609288f37ab6927b86df519e38d4", 
    "dest": "/home/vagrant/0_README.txt", 
    "elapsed": 0, 
    "gid": 1000, 
    "group": "vagrant", 
    "md5sum": "535a06e83328bf3b18e0afd6523069c8", 
    "mode": "0664", 
    "msg": "OK (2495 bytes)", 
    "owner": "vagrant", 
    "secontext": "unconfined_u:object_r:user_home_t:s0", 
    "size": 2495, 
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1659215955.74-23346-177381045837174/tmpep4xRe", 
    "state": "file", 
    "status_code": 200, 
    "uid": 1000, 
    "url": "https://mirror.yandex.ru/centos/7.9.2009/isos/x86_64/0_README.txt"
}
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum_dest": null, 
    "checksum_src": "87c0d9dc6757609288f37ab6927b86df519e38d4", 
    "dest": "/home/vagrant/0_README.txt", 
    "elapsed": 0, 
    "gid": 1000, 
    "group": "vagrant", 
    "md5sum": "535a06e83328bf3b18e0afd6523069c8", 
    "mode": "0664", 
    "msg": "OK (2495 bytes)", 
    "owner": "vagrant", 
    "secontext": "unconfined_u:object_r:user_home_t:s0", 
    "size": 2495, 
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1659215955.78-23348-64415116074640/tmpJ5CCQ5", 
    "state": "file", 
    "status_code": 200, 
    "uid": 1000, 
    "url": "https://mirror.yandex.ru/centos/7.9.2009/isos/x86_64/0_README.txt"
}
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "checksum_dest": null, 
    "checksum_src": "87c0d9dc6757609288f37ab6927b86df519e38d4", 
    "dest": "/home/vagrant/0_README.txt", 
    "elapsed": 0, 
    "gid": 1000, 
    "group": "vagrant", 
    "md5sum": "535a06e83328bf3b18e0afd6523069c8", 
    "mode": "0664", 
    "msg": "OK (2495 bytes)", 
    "owner": "vagrant", 
    "secontext": "unconfined_u:object_r:user_home_t:s0", 
    "size": 2495, 
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1659215955.69-23350-99548286681631/tmpm9RXkX", 
    "state": "file", 
    "status_code": 200, 
    "uid": 1000, 
    "url": "https://mirror.yandex.ru/centos/7.9.2009/isos/x86_64/0_README.txt"
}
[user@localhost ansible]$
...

Проверим, что файл скачался:
...
[user@localhost ansible]$ ansible all -m shell -a "ls -l /home/vagrant"
host1 | CHANGED | rc=0 >>
total 4
-rw-rw-r--. 1 vagrant vagrant 2495 Jul 30 21:19 0_README.txt
host2 | CHANGED | rc=0 >>
total 4
-rw-rw-r--. 1 vagrant vagrant 2495 Jul 30 21:19 0_README.txt
host0 | CHANGED | rc=0 >>
total 4
-rw-rw-r--. 1 vagrant vagrant 2495 Jul 30 21:19 0_README.txt
[user@localhost ansible]$
...
Как видим, что файл 0_README.txt скачался с интернета в домашнюю директорию пользователя vagrant.
Удалим этот файл:
...
[user@localhost ansible]$ ansible all -m file -a "path=/home/vagrant/0_README.txt state=absent"
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "path": "/home/vagrant/0_README.txt", 
    "state": "absent"
}
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "path": "/home/vagrant/0_README.txt", 
    "state": "absent"
}
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "path": "/home/vagrant/0_README.txt", 
    "state": "absent"
}
[user@localhost ansible]$
...
Удаление файла 0_README.txt прошло успешно.

Установим какую-нибудь программу на всех серверах, например, httpd:
...
[user@localhost ansible]$ ansible all -m yum -a "name=httpd state=installed" -b
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "installed": [
            "httpd"
        ]
    }, 
    "msg": "warning: /var/cache/yum/x86_64/7/base/packages/apr-1.4.8-7.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY\nImporting GPG key 0xF4A80EB5:\n Userid     : \"CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>\"\n Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5\n Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)\n From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7\n", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nLoading mirror speeds from cached hostfile\n * base: mirror.surf\n * extras: mirror.surf\n * updates: mirror.surf\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n--> Processing Dependency: httpd-tools = 2.4.6-97.el7.centos.5 for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: system-logos >= 7.92.1-1 for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: /etc/mime.types for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Running transaction check\n---> Package apr.x86_64 0:1.4.8-7.el7 will be installed\n---> Package apr-util.x86_64 0:1.5.2-6.el7 will be installed\n---> Package centos-logos.noarch 0:70.0.6-3.el7.centos will be installed\n---> Package httpd-tools.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n---> Package mailcap.noarch 0:2.1.41-2.el7 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package           Arch        Version                       Repository    Size\n================================================================================\nInstalling:\n httpd             x86_64      2.4.6-97.el7.centos.5         updates      2.7 M\nInstalling for dependencies:\n apr               x86_64      1.4.8-7.el7                   base         104 k\n apr-util          x86_64      1.5.2-6.el7                   base          92 k\n centos-logos      noarch      70.0.6-3.el7.centos           base          21 M\n httpd-tools       x86_64      2.4.6-97.el7.centos.5         updates       94 k\n mailcap           noarch      2.1.41-2.el7                  base          31 k\n\nTransaction Summary\n================================================================================\nInstall  1 Package (+5 Dependent packages)\n\nTotal download size: 24 M\nInstalled size: 32 M\nDownloading packages:\nPublic key for apr-1.4.8-7.el7.x86_64.rpm is not installed\nPublic key for httpd-tools-2.4.6-97.el7.centos.5.x86_64.rpm is not installed\n--------------------------------------------------------------------------------\nTotal                                              2.7 MB/s |  24 MB  00:09     \nRetrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : apr-1.4.8-7.el7.x86_64                                       1/6 \n  Installing : apr-util-1.5.2-6.el7.x86_64                                  2/6 \n  Installing : httpd-tools-2.4.6-97.el7.centos.5.x86_64                     3/6 \n  Installing : centos-logos-70.0.6-3.el7.centos.noarch                      4/6 \n  Installing : mailcap-2.1.41-2.el7.noarch                                  5/6 \n  Installing : httpd-2.4.6-97.el7.centos.5.x86_64                           6/6 \n  Verifying  : httpd-tools-2.4.6-97.el7.centos.5.x86_64                     1/6 \n  Verifying  : mailcap-2.1.41-2.el7.noarch                                  2/6 \n  Verifying  : apr-1.4.8-7.el7.x86_64                                       3/6 \n  Verifying  : apr-util-1.5.2-6.el7.x86_64                                  4/6 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           5/6 \n  Verifying  : centos-logos-70.0.6-3.el7.centos.noarch                      6/6 \n\nInstalled:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nDependency Installed:\n  apr.x86_64 0:1.4.8-7.el7                                                      \n  apr-util.x86_64 0:1.5.2-6.el7                                                 \n  centos-logos.noarch 0:70.0.6-3.el7.centos                                     \n  httpd-tools.x86_64 0:2.4.6-97.el7.centos.5                                    \n  mailcap.noarch 0:2.1.41-2.el7                                                 \n\nComplete!\n"
    ]
}
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "installed": [
            "httpd"
        ]
    }, 
    "msg": "warning: /var/cache/yum/x86_64/7/base/packages/apr-1.4.8-7.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY\nImporting GPG key 0xF4A80EB5:\n Userid     : \"CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>\"\n Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5\n Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)\n From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7\n", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nLoading mirror speeds from cached hostfile\n * base: mirror.surf\n * extras: mirror.surf\n * updates: mirror.surf\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n--> Processing Dependency: httpd-tools = 2.4.6-97.el7.centos.5 for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: system-logos >= 7.92.1-1 for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: /etc/mime.types for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Running transaction check\n---> Package apr.x86_64 0:1.4.8-7.el7 will be installed\n---> Package apr-util.x86_64 0:1.5.2-6.el7 will be installed\n---> Package centos-logos.noarch 0:70.0.6-3.el7.centos will be installed\n---> Package httpd-tools.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n---> Package mailcap.noarch 0:2.1.41-2.el7 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package           Arch        Version                       Repository    Size\n================================================================================\nInstalling:\n httpd             x86_64      2.4.6-97.el7.centos.5         updates      2.7 M\nInstalling for dependencies:\n apr               x86_64      1.4.8-7.el7                   base         104 k\n apr-util          x86_64      1.5.2-6.el7                   base          92 k\n centos-logos      noarch      70.0.6-3.el7.centos           base          21 M\n httpd-tools       x86_64      2.4.6-97.el7.centos.5         updates       94 k\n mailcap           noarch      2.1.41-2.el7                  base          31 k\n\nTransaction Summary\n================================================================================\nInstall  1 Package (+5 Dependent packages)\n\nTotal download size: 24 M\nInstalled size: 32 M\nDownloading packages:\nPublic key for apr-1.4.8-7.el7.x86_64.rpm is not installed\nPublic key for httpd-tools-2.4.6-97.el7.centos.5.x86_64.rpm is not installed\n--------------------------------------------------------------------------------\nTotal                                              1.9 MB/s |  24 MB  00:13     \nRetrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : apr-1.4.8-7.el7.x86_64                                       1/6 \n  Installing : apr-util-1.5.2-6.el7.x86_64                                  2/6 \n  Installing : httpd-tools-2.4.6-97.el7.centos.5.x86_64                     3/6 \n  Installing : centos-logos-70.0.6-3.el7.centos.noarch                      4/6 \n  Installing : mailcap-2.1.41-2.el7.noarch                                  5/6 \n  Installing : httpd-2.4.6-97.el7.centos.5.x86_64                           6/6 \n  Verifying  : httpd-tools-2.4.6-97.el7.centos.5.x86_64                     1/6 \n  Verifying  : mailcap-2.1.41-2.el7.noarch                                  2/6 \n  Verifying  : apr-1.4.8-7.el7.x86_64                                       3/6 \n  Verifying  : apr-util-1.5.2-6.el7.x86_64                                  4/6 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           5/6 \n  Verifying  : centos-logos-70.0.6-3.el7.centos.noarch                      6/6 \n\nInstalled:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nDependency Installed:\n  apr.x86_64 0:1.4.8-7.el7                                                      \n  apr-util.x86_64 0:1.5.2-6.el7                                                 \n  centos-logos.noarch 0:70.0.6-3.el7.centos                                     \n  httpd-tools.x86_64 0:2.4.6-97.el7.centos.5                                    \n  mailcap.noarch 0:2.1.41-2.el7                                                 \n\nComplete!\n"
    ]
}
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "installed": [
            "httpd"
        ]
    }, 
    "msg": "warning: /var/cache/yum/x86_64/7/base/packages/apr-util-1.5.2-6.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY\nImporting GPG key 0xF4A80EB5:\n Userid     : \"CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>\"\n Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5\n Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)\n From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7\n", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nLoading mirror speeds from cached hostfile\n * base: mirror.surf\n * extras: mirror.surf\n * updates: mirror.surf\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n--> Processing Dependency: httpd-tools = 2.4.6-97.el7.centos.5 for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: system-logos >= 7.92.1-1 for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: /etc/mime.types for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-2.4.6-97.el7.centos.5.x86_64\n--> Running transaction check\n---> Package apr.x86_64 0:1.4.8-7.el7 will be installed\n---> Package apr-util.x86_64 0:1.5.2-6.el7 will be installed\n---> Package centos-logos.noarch 0:70.0.6-3.el7.centos will be installed\n---> Package httpd-tools.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n---> Package mailcap.noarch 0:2.1.41-2.el7 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package           Arch        Version                       Repository    Size\n================================================================================\nInstalling:\n httpd             x86_64      2.4.6-97.el7.centos.5         updates      2.7 M\nInstalling for dependencies:\n apr               x86_64      1.4.8-7.el7                   base         104 k\n apr-util          x86_64      1.5.2-6.el7                   base          92 k\n centos-logos      noarch      70.0.6-3.el7.centos           base          21 M\n httpd-tools       x86_64      2.4.6-97.el7.centos.5         updates       94 k\n mailcap           noarch      2.1.41-2.el7                  base          31 k\n\nTransaction Summary\n================================================================================\nInstall  1 Package (+5 Dependent packages)\n\nTotal download size: 24 M\nInstalled size: 32 M\nDownloading packages:\nPublic key for apr-util-1.5.2-6.el7.x86_64.rpm is not installed\nPublic key for httpd-tools-2.4.6-97.el7.centos.5.x86_64.rpm is not installed\n--------------------------------------------------------------------------------\nTotal                                              1.3 MB/s |  24 MB  00:18     \nRetrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : apr-1.4.8-7.el7.x86_64                                       1/6 \n  Installing : apr-util-1.5.2-6.el7.x86_64                                  2/6 \n  Installing : httpd-tools-2.4.6-97.el7.centos.5.x86_64                     3/6 \n  Installing : centos-logos-70.0.6-3.el7.centos.noarch                      4/6 \n  Installing : mailcap-2.1.41-2.el7.noarch                                  5/6 \n  Installing : httpd-2.4.6-97.el7.centos.5.x86_64                           6/6 \n  Verifying  : httpd-tools-2.4.6-97.el7.centos.5.x86_64                     1/6 \n  Verifying  : mailcap-2.1.41-2.el7.noarch                                  2/6 \n  Verifying  : apr-1.4.8-7.el7.x86_64                                       3/6 \n  Verifying  : apr-util-1.5.2-6.el7.x86_64                                  4/6 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           5/6 \n  Verifying  : centos-logos-70.0.6-3.el7.centos.noarch                      6/6 \n\nInstalled:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nDependency Installed:\n  apr.x86_64 0:1.4.8-7.el7                                                      \n  apr-util.x86_64 0:1.5.2-6.el7                                                 \n  centos-logos.noarch 0:70.0.6-3.el7.centos                                     \n  httpd-tools.x86_64 0:2.4.6-97.el7.centos.5                                    \n  mailcap.noarch 0:2.1.41-2.el7                                                 \n\nComplete!\n"
    ]
}
[user@localhost ansible]$
...

Теперь удалим программу httpd:
...
[user@localhost ansible]$ ansible all -m yum -a "name=httpd state=removed" -b
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
[user@localhost ansible]$
...

Прочитаем с каждого сервера какой-нибудь сайт, например, http://www.adv-it.net, где return_content=yes - прочитать сам контент сайта:
...
[user@localhost ansible]$ ansible all -m uri -a "url=http://www.adv-it.net return_content=yes"
host1 | SUCCESS => {
    "accept_ranges": "bytes", 
    "age": "58706", 
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "connection": "close", 
    "content": "\r\n<HTML>\r\n<HEAD>\r\n<TITLE>ADV-IT</TITLE>\r\n<SCRIPT LANGUAGE=\"JavaScript\">\r\n      var sizes = new Array(0,1,2,4,8,10,12);\r\n      sizes.pos = 0;\r\n    \r\nfunction Elastic()\r\n{\r\n    var el = document.all.elastic\r\n    if (null == el.direction)el.direction = 1\r\n    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))\r\n    el.direction *= -1\r\n    el.style.letterSpacing = sizes[sizes.pos += el.direction]\r\nsetTimeout('Elastic()',100)\r\n}\r\n\r\n</SCRIPT>\r\n<BODY  background=\"kamni.jpg\"  onLoad=Elastic()>\r\n<CENTER>\r\n<font color=\"white\"><h2>Welcome to my World!</h2>\r\n<font color=\"yellow\"><H1 ID=\"elastic\" ALIGN=\"Center\">ADV-IT</H1>\r\n<br>\r\n<img src=\"image2.jpg\">\r\n<img src=\"image1.jpg\"><br>\r\n\r\n</body>\r\n</HTML>", 
    "content_length": "704", 
    "content_type": "text/html", 
    "cookies": {}, 
    "cookies_string": "", 
    "date": "Sat, 30 Jul 2022 05:34:10 GMT", 
    "elapsed": 0, 
    "etag": "\"822d766c69162be9e05b656f19e6194d\"", 
    "last_modified": "Sun, 21 Jan 2018 01:00:05 GMT", 
    "msg": "OK (704 bytes)", 
    "redirected": true, 
    "server": "AmazonS3", 
    "status": 200, 
    "url": "https://www.adv-it.net/", 
    "via": "1.1 d8b0b3928e53502c6ce822abc3cc3d70.cloudfront.net (CloudFront)", 
    "x_amz_cf_id": "SXXuMNnagNigKZIsOHKZ-bJv68Sj5IpuMc1aVmrClFJ75hQde3UCMg==", 
    "x_amz_cf_pop": "HEL51-P1", 
    "x_cache": "Hit from cloudfront"
}
host2 | SUCCESS => {
    "accept_ranges": "bytes", 
    "age": "58706", 
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "connection": "close", 
    "content": "\r\n<HTML>\r\n<HEAD>\r\n<TITLE>ADV-IT</TITLE>\r\n<SCRIPT LANGUAGE=\"JavaScript\">\r\n      var sizes = new Array(0,1,2,4,8,10,12);\r\n      sizes.pos = 0;\r\n    \r\nfunction Elastic()\r\n{\r\n    var el = document.all.elastic\r\n    if (null == el.direction)el.direction = 1\r\n    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))\r\n    el.direction *= -1\r\n    el.style.letterSpacing = sizes[sizes.pos += el.direction]\r\nsetTimeout('Elastic()',100)\r\n}\r\n\r\n</SCRIPT>\r\n<BODY  background=\"kamni.jpg\"  onLoad=Elastic()>\r\n<CENTER>\r\n<font color=\"white\"><h2>Welcome to my World!</h2>\r\n<font color=\"yellow\"><H1 ID=\"elastic\" ALIGN=\"Center\">ADV-IT</H1>\r\n<br>\r\n<img src=\"image2.jpg\">\r\n<img src=\"image1.jpg\"><br>\r\n\r\n</body>\r\n</HTML>", 
    "content_length": "704", 
    "content_type": "text/html", 
    "cookies": {}, 
    "cookies_string": "", 
    "date": "Sat, 30 Jul 2022 05:34:10 GMT", 
    "elapsed": 0, 
    "etag": "\"822d766c69162be9e05b656f19e6194d\"", 
    "last_modified": "Sun, 21 Jan 2018 01:00:05 GMT", 
    "msg": "OK (704 bytes)", 
    "redirected": true, 
    "server": "AmazonS3", 
    "status": 200, 
    "url": "https://www.adv-it.net/", 
    "via": "1.1 1c104af9dcb33e29b8c5ed9ebabafb86.cloudfront.net (CloudFront)", 
    "x_amz_cf_id": "3B9bF5T5iRQsVBUKFLcS1j1sJAaGqVp657x8zPdXpPQ_GWf4gd8a7g==", 
    "x_amz_cf_pop": "HEL51-P1", 
    "x_cache": "Hit from cloudfront"
}
host0 | SUCCESS => {
    "accept_ranges": "bytes", 
    "age": "58706", 
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": false, 
    "connection": "close", 
    "content": "\r\n<HTML>\r\n<HEAD>\r\n<TITLE>ADV-IT</TITLE>\r\n<SCRIPT LANGUAGE=\"JavaScript\">\r\n      var sizes = new Array(0,1,2,4,8,10,12);\r\n      sizes.pos = 0;\r\n    \r\nfunction Elastic()\r\n{\r\n    var el = document.all.elastic\r\n    if (null == el.direction)el.direction = 1\r\n    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))\r\n    el.direction *= -1\r\n    el.style.letterSpacing = sizes[sizes.pos += el.direction]\r\nsetTimeout('Elastic()',100)\r\n}\r\n\r\n</SCRIPT>\r\n<BODY  background=\"kamni.jpg\"  onLoad=Elastic()>\r\n<CENTER>\r\n<font color=\"white\"><h2>Welcome to my World!</h2>\r\n<font color=\"yellow\"><H1 ID=\"elastic\" ALIGN=\"Center\">ADV-IT</H1>\r\n<br>\r\n<img src=\"image2.jpg\">\r\n<img src=\"image1.jpg\"><br>\r\n\r\n</body>\r\n</HTML>", 
    "content_length": "704", 
    "content_type": "text/html", 
    "cookies": {}, 
    "cookies_string": "", 
    "date": "Sat, 30 Jul 2022 05:34:10 GMT", 
    "elapsed": 0, 
    "etag": "\"822d766c69162be9e05b656f19e6194d\"", 
    "last_modified": "Sun, 21 Jan 2018 01:00:05 GMT", 
    "msg": "OK (704 bytes)", 
    "redirected": true, 
    "server": "AmazonS3", 
    "status": 200, 
    "url": "https://www.adv-it.net/", 
    "via": "1.1 1be5216f770ec05deb91e9e25b61b898.cloudfront.net (CloudFront)", 
    "x_amz_cf_id": "fkQ8EM7hRDlZl9ReNvGKD-02__j13IthHTu115SN7cO0tZKLkeOQAQ==", 
    "x_amz_cf_pop": "HEL51-P1", 
    "x_cache": "Hit from cloudfront"
}
[user@localhost ansible]$
...

Ещё раз установим httpd, вместо state=installed можно использовать state=latest:
...
[user@localhost ansible]$ ansible all -m yum -a "name=httpd state=latest" -b
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "installed": [
            "httpd"
        ], 
        "updated": []
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nLoading mirror speeds from cached hostfile\n * base: mirror.surf\n * extras: mirror.surf\n * updates: mirror.surf\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                        Repository      Size\n================================================================================\nInstalling:\n httpd        x86_64        2.4.6-97.el7.centos.5          updates        2.7 M\n\nTransaction Summary\n================================================================================\nInstall  1 Package\n\nTotal download size: 2.7 M\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nInstalled:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "installed": [
            "httpd"
        ], 
        "updated": []
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nLoading mirror speeds from cached hostfile\n * base: mirror.surf\n * extras: mirror.surf\n * updates: mirror.surf\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                        Repository      Size\n================================================================================\nInstalling:\n httpd        x86_64        2.4.6-97.el7.centos.5          updates        2.7 M\n\nTransaction Summary\n================================================================================\nInstall  1 Package\n\nTotal download size: 2.7 M\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nInstalled:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "installed": [
            "httpd"
        ], 
        "updated": []
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nLoading mirror speeds from cached hostfile\n * base: mirror.surf\n * extras: mirror.surf\n * updates: mirror.surf\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be installed\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                        Repository      Size\n================================================================================\nInstalling:\n httpd        x86_64        2.4.6-97.el7.centos.5          updates        2.7 M\n\nTransaction Summary\n================================================================================\nInstall  1 Package\n\nTotal download size: 2.7 M\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nInstalled:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
[user@localhost ansible]$
...

Теперь запустим сервис httpd и установим автостарт после установки:
...
[user@localhost ansible]$ ansible all -m service -a "name=httpd state=started enabled=yes" -b
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "enabled": true, 
    "name": "httpd", 
    "state": "started", 
    "status": {
        "ActiveEnterTimestampMonotonic": "0", 
        "ActiveExitTimestampMonotonic": "0", 
        "ActiveState": "inactive", 
        "After": "systemd-journald.socket basic.target -.mount tmp.mount remote-fs.target system.slice network.target nss-lookup.target", 
        "AllowIsolate": "no", 
        "AmbientCapabilities": "0", 
        "AssertResult": "no", 
        "AssertTimestampMonotonic": "0", 
        "Before": "shutdown.target", 
        "BlockIOAccounting": "no", 
        "BlockIOWeight": "18446744073709551615", 
        "CPUAccounting": "no", 
        "CPUQuotaPerSecUSec": "infinity", 
        "CPUSchedulingPolicy": "0", 
        "CPUSchedulingPriority": "0", 
        "CPUSchedulingResetOnFork": "no", 
        "CPUShares": "18446744073709551615", 
        "CanIsolate": "no", 
        "CanReload": "yes", 
        "CanStart": "yes", 
        "CanStop": "yes", 
        "CapabilityBoundingSet": "18446744073709551615", 
        "ConditionResult": "no", 
        "ConditionTimestampMonotonic": "0", 
        "Conflicts": "shutdown.target", 
        "ControlPID": "0", 
        "DefaultDependencies": "yes", 
        "Delegate": "no", 
        "Description": "The Apache HTTP Server", 
        "DevicePolicy": "auto", 
        "Documentation": "man:httpd(8) man:apachectl(8)", 
        "EnvironmentFile": "/etc/sysconfig/httpd (ignore_errors=no)", 
        "ExecMainCode": "0", 
        "ExecMainExitTimestampMonotonic": "0", 
        "ExecMainPID": "0", 
        "ExecMainStartTimestampMonotonic": "0", 
        "ExecMainStatus": "0", 
        "ExecReload": "{ path=/usr/sbin/httpd ; argv[]=/usr/sbin/httpd $OPTIONS -k graceful ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "ExecStart": "{ path=/usr/sbin/httpd ; argv[]=/usr/sbin/httpd $OPTIONS -DFOREGROUND ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "ExecStop": "{ path=/bin/kill ; argv[]=/bin/kill -WINCH ${MAINPID} ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "FailureAction": "none", 
        "FileDescriptorStoreMax": "0", 
        "FragmentPath": "/usr/lib/systemd/system/httpd.service", 
        "GuessMainPID": "yes", 
        "IOScheduling": "0", 
        "Id": "httpd.service", 
        "IgnoreOnIsolate": "no", 
        "IgnoreOnSnapshot": "no", 
        "IgnoreSIGPIPE": "yes", 
        "InactiveEnterTimestampMonotonic": "0", 
        "InactiveExitTimestampMonotonic": "0", 
        "JobTimeoutAction": "none", 
        "JobTimeoutUSec": "0", 
        "KillMode": "control-group", 
        "KillSignal": "18", 
        "LimitAS": "18446744073709551615", 
        "LimitCORE": "18446744073709551615", 
        "LimitCPU": "18446744073709551615", 
        "LimitDATA": "18446744073709551615", 
        "LimitFSIZE": "18446744073709551615", 
        "LimitLOCKS": "18446744073709551615", 
        "LimitMEMLOCK": "65536", 
        "LimitMSGQUEUE": "819200", 
        "LimitNICE": "0", 
        "LimitNOFILE": "4096", 
        "LimitNPROC": "881", 
        "LimitRSS": "18446744073709551615", 
        "LimitRTPRIO": "0", 
        "LimitRTTIME": "18446744073709551615", 
        "LimitSIGPENDING": "881", 
        "LimitSTACK": "18446744073709551615", 
        "LoadState": "loaded", 
        "MainPID": "0", 
        "MemoryAccounting": "no", 
        "MemoryCurrent": "18446744073709551615", 
        "MemoryLimit": "18446744073709551615", 
        "MountFlags": "0", 
        "Names": "httpd.service", 
        "NeedDaemonReload": "no", 
        "Nice": "0", 
        "NoNewPrivileges": "no", 
        "NonBlocking": "no", 
        "NotifyAccess": "main", 
        "OOMScoreAdjust": "0", 
        "OnFailureJobMode": "replace", 
        "PermissionsStartOnly": "no", 
        "PrivateDevices": "no", 
        "PrivateNetwork": "no", 
        "PrivateTmp": "yes", 
        "ProtectHome": "no", 
        "ProtectSystem": "no", 
        "RefuseManualStart": "no", 
        "RefuseManualStop": "no", 
        "RemainAfterExit": "no", 
        "Requires": "system.slice basic.target -.mount", 
        "RequiresMountsFor": "/var/tmp", 
        "Restart": "no", 
        "RestartUSec": "100ms", 
        "Result": "success", 
        "RootDirectoryStartOnly": "no", 
        "RuntimeDirectoryMode": "0755", 
        "SameProcessGroup": "no", 
        "SecureBits": "0", 
        "SendSIGHUP": "no", 
        "SendSIGKILL": "yes", 
        "Slice": "system.slice", 
        "StandardError": "inherit", 
        "StandardInput": "null", 
        "StandardOutput": "journal", 
        "StartLimitAction": "none", 
        "StartLimitBurst": "5", 
        "StartLimitInterval": "10000000", 
        "StartupBlockIOWeight": "18446744073709551615", 
        "StartupCPUShares": "18446744073709551615", 
        "StatusErrno": "0", 
        "StopWhenUnneeded": "no", 
        "SubState": "dead", 
        "SyslogLevelPrefix": "yes", 
        "SyslogPriority": "30", 
        "SystemCallErrorNumber": "0", 
        "TTYReset": "no", 
        "TTYVHangup": "no", 
        "TTYVTDisallocate": "no", 
        "TasksAccounting": "no", 
        "TasksCurrent": "18446744073709551615", 
        "TasksMax": "18446744073709551615", 
        "TimeoutStartUSec": "1min 30s", 
        "TimeoutStopUSec": "1min 30s", 
        "TimerSlackNSec": "50000", 
        "Transient": "no", 
        "Type": "notify", 
        "UMask": "0022", 
        "UnitFilePreset": "disabled", 
        "UnitFileState": "disabled", 
        "WatchdogTimestampMonotonic": "0", 
        "WatchdogUSec": "0"
    }
}
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "enabled": true, 
    "name": "httpd", 
    "state": "started", 
    "status": {
        "ActiveEnterTimestampMonotonic": "0", 
        "ActiveExitTimestampMonotonic": "0", 
        "ActiveState": "inactive", 
        "After": "system.slice nss-lookup.target remote-fs.target systemd-journald.socket tmp.mount network.target basic.target -.mount", 
        "AllowIsolate": "no", 
        "AmbientCapabilities": "0", 
        "AssertResult": "no", 
        "AssertTimestampMonotonic": "0", 
        "Before": "shutdown.target", 
        "BlockIOAccounting": "no", 
        "BlockIOWeight": "18446744073709551615", 
        "CPUAccounting": "no", 
        "CPUQuotaPerSecUSec": "infinity", 
        "CPUSchedulingPolicy": "0", 
        "CPUSchedulingPriority": "0", 
        "CPUSchedulingResetOnFork": "no", 
        "CPUShares": "18446744073709551615", 
        "CanIsolate": "no", 
        "CanReload": "yes", 
        "CanStart": "yes", 
        "CanStop": "yes", 
        "CapabilityBoundingSet": "18446744073709551615", 
        "ConditionResult": "no", 
        "ConditionTimestampMonotonic": "0", 
        "Conflicts": "shutdown.target", 
        "ControlPID": "0", 
        "DefaultDependencies": "yes", 
        "Delegate": "no", 
        "Description": "The Apache HTTP Server", 
        "DevicePolicy": "auto", 
        "Documentation": "man:httpd(8) man:apachectl(8)", 
        "EnvironmentFile": "/etc/sysconfig/httpd (ignore_errors=no)", 
        "ExecMainCode": "0", 
        "ExecMainExitTimestampMonotonic": "0", 
        "ExecMainPID": "0", 
        "ExecMainStartTimestampMonotonic": "0", 
        "ExecMainStatus": "0", 
        "ExecReload": "{ path=/usr/sbin/httpd ; argv[]=/usr/sbin/httpd $OPTIONS -k graceful ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "ExecStart": "{ path=/usr/sbin/httpd ; argv[]=/usr/sbin/httpd $OPTIONS -DFOREGROUND ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "ExecStop": "{ path=/bin/kill ; argv[]=/bin/kill -WINCH ${MAINPID} ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "FailureAction": "none", 
        "FileDescriptorStoreMax": "0", 
        "FragmentPath": "/usr/lib/systemd/system/httpd.service", 
        "GuessMainPID": "yes", 
        "IOScheduling": "0", 
        "Id": "httpd.service", 
        "IgnoreOnIsolate": "no", 
        "IgnoreOnSnapshot": "no", 
        "IgnoreSIGPIPE": "yes", 
        "InactiveEnterTimestampMonotonic": "0", 
        "InactiveExitTimestampMonotonic": "0", 
        "JobTimeoutAction": "none", 
        "JobTimeoutUSec": "0", 
        "KillMode": "control-group", 
        "KillSignal": "18", 
        "LimitAS": "18446744073709551615", 
        "LimitCORE": "18446744073709551615", 
        "LimitCPU": "18446744073709551615", 
        "LimitDATA": "18446744073709551615", 
        "LimitFSIZE": "18446744073709551615", 
        "LimitLOCKS": "18446744073709551615", 
        "LimitMEMLOCK": "65536", 
        "LimitMSGQUEUE": "819200", 
        "LimitNICE": "0", 
        "LimitNOFILE": "4096", 
        "LimitNPROC": "881", 
        "LimitRSS": "18446744073709551615", 
        "LimitRTPRIO": "0", 
        "LimitRTTIME": "18446744073709551615", 
        "LimitSIGPENDING": "881", 
        "LimitSTACK": "18446744073709551615", 
        "LoadState": "loaded", 
        "MainPID": "0", 
        "MemoryAccounting": "no", 
        "MemoryCurrent": "18446744073709551615", 
        "MemoryLimit": "18446744073709551615", 
        "MountFlags": "0", 
        "Names": "httpd.service", 
        "NeedDaemonReload": "no", 
        "Nice": "0", 
        "NoNewPrivileges": "no", 
        "NonBlocking": "no", 
        "NotifyAccess": "main", 
        "OOMScoreAdjust": "0", 
        "OnFailureJobMode": "replace", 
        "PermissionsStartOnly": "no", 
        "PrivateDevices": "no", 
        "PrivateNetwork": "no", 
        "PrivateTmp": "yes", 
        "ProtectHome": "no", 
        "ProtectSystem": "no", 
        "RefuseManualStart": "no", 
        "RefuseManualStop": "no", 
        "RemainAfterExit": "no", 
        "Requires": "-.mount basic.target system.slice", 
        "RequiresMountsFor": "/var/tmp", 
        "Restart": "no", 
        "RestartUSec": "100ms", 
        "Result": "success", 
        "RootDirectoryStartOnly": "no", 
        "RuntimeDirectoryMode": "0755", 
        "SameProcessGroup": "no", 
        "SecureBits": "0", 
        "SendSIGHUP": "no", 
        "SendSIGKILL": "yes", 
        "Slice": "system.slice", 
        "StandardError": "inherit", 
        "StandardInput": "null", 
        "StandardOutput": "journal", 
        "StartLimitAction": "none", 
        "StartLimitBurst": "5", 
        "StartLimitInterval": "10000000", 
        "StartupBlockIOWeight": "18446744073709551615", 
        "StartupCPUShares": "18446744073709551615", 
        "StatusErrno": "0", 
        "StopWhenUnneeded": "no", 
        "SubState": "dead", 
        "SyslogLevelPrefix": "yes", 
        "SyslogPriority": "30", 
        "SystemCallErrorNumber": "0", 
        "TTYReset": "no", 
        "TTYVHangup": "no", 
        "TTYVTDisallocate": "no", 
        "TasksAccounting": "no", 
        "TasksCurrent": "18446744073709551615", 
        "TasksMax": "18446744073709551615", 
        "TimeoutStartUSec": "1min 30s", 
        "TimeoutStopUSec": "1min 30s", 
        "TimerSlackNSec": "50000", 
        "Transient": "no", 
        "Type": "notify", 
        "UMask": "0022", 
        "UnitFilePreset": "disabled", 
        "UnitFileState": "disabled", 
        "WatchdogTimestampMonotonic": "0", 
        "WatchdogUSec": "0"
    }
}
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "enabled": true, 
    "name": "httpd", 
    "state": "started", 
    "status": {
        "ActiveEnterTimestampMonotonic": "0", 
        "ActiveExitTimestampMonotonic": "0", 
        "ActiveState": "inactive", 
        "After": "network.target system.slice remote-fs.target systemd-journald.socket tmp.mount nss-lookup.target basic.target -.mount", 
        "AllowIsolate": "no", 
        "AmbientCapabilities": "0", 
        "AssertResult": "no", 
        "AssertTimestampMonotonic": "0", 
        "Before": "shutdown.target", 
        "BlockIOAccounting": "no", 
        "BlockIOWeight": "18446744073709551615", 
        "CPUAccounting": "no", 
        "CPUQuotaPerSecUSec": "infinity", 
        "CPUSchedulingPolicy": "0", 
        "CPUSchedulingPriority": "0", 
        "CPUSchedulingResetOnFork": "no", 
        "CPUShares": "18446744073709551615", 
        "CanIsolate": "no", 
        "CanReload": "yes", 
        "CanStart": "yes", 
        "CanStop": "yes", 
        "CapabilityBoundingSet": "18446744073709551615", 
        "ConditionResult": "no", 
        "ConditionTimestampMonotonic": "0", 
        "Conflicts": "shutdown.target", 
        "ControlPID": "0", 
        "DefaultDependencies": "yes", 
        "Delegate": "no", 
        "Description": "The Apache HTTP Server", 
        "DevicePolicy": "auto", 
        "Documentation": "man:httpd(8) man:apachectl(8)", 
        "EnvironmentFile": "/etc/sysconfig/httpd (ignore_errors=no)", 
        "ExecMainCode": "0", 
        "ExecMainExitTimestampMonotonic": "0", 
        "ExecMainPID": "0", 
        "ExecMainStartTimestampMonotonic": "0", 
        "ExecMainStatus": "0", 
        "ExecReload": "{ path=/usr/sbin/httpd ; argv[]=/usr/sbin/httpd $OPTIONS -k graceful ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "ExecStart": "{ path=/usr/sbin/httpd ; argv[]=/usr/sbin/httpd $OPTIONS -DFOREGROUND ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "ExecStop": "{ path=/bin/kill ; argv[]=/bin/kill -WINCH ${MAINPID} ; ignore_errors=no ; start_time=[n/a] ; stop_time=[n/a] ; pid=0 ; code=(null) ; status=0/0 }", 
        "FailureAction": "none", 
        "FileDescriptorStoreMax": "0", 
        "FragmentPath": "/usr/lib/systemd/system/httpd.service", 
        "GuessMainPID": "yes", 
        "IOScheduling": "0", 
        "Id": "httpd.service", 
        "IgnoreOnIsolate": "no", 
        "IgnoreOnSnapshot": "no", 
        "IgnoreSIGPIPE": "yes", 
        "InactiveEnterTimestampMonotonic": "0", 
        "InactiveExitTimestampMonotonic": "0", 
        "JobTimeoutAction": "none", 
        "JobTimeoutUSec": "0", 
        "KillMode": "control-group", 
        "KillSignal": "18", 
        "LimitAS": "18446744073709551615", 
        "LimitCORE": "18446744073709551615", 
        "LimitCPU": "18446744073709551615", 
        "LimitDATA": "18446744073709551615", 
        "LimitFSIZE": "18446744073709551615", 
        "LimitLOCKS": "18446744073709551615", 
        "LimitMEMLOCK": "65536", 
        "LimitMSGQUEUE": "819200", 
        "LimitNICE": "0", 
        "LimitNOFILE": "4096", 
        "LimitNPROC": "881", 
        "LimitRSS": "18446744073709551615", 
        "LimitRTPRIO": "0", 
        "LimitRTTIME": "18446744073709551615", 
        "LimitSIGPENDING": "881", 
        "LimitSTACK": "18446744073709551615", 
        "LoadState": "loaded", 
        "MainPID": "0", 
        "MemoryAccounting": "no", 
        "MemoryCurrent": "18446744073709551615", 
        "MemoryLimit": "18446744073709551615", 
        "MountFlags": "0", 
        "Names": "httpd.service", 
        "NeedDaemonReload": "no", 
        "Nice": "0", 
        "NoNewPrivileges": "no", 
        "NonBlocking": "no", 
        "NotifyAccess": "main", 
        "OOMScoreAdjust": "0", 
        "OnFailureJobMode": "replace", 
        "PermissionsStartOnly": "no", 
        "PrivateDevices": "no", 
        "PrivateNetwork": "no", 
        "PrivateTmp": "yes", 
        "ProtectHome": "no", 
        "ProtectSystem": "no", 
        "RefuseManualStart": "no", 
        "RefuseManualStop": "no", 
        "RemainAfterExit": "no", 
        "Requires": "basic.target -.mount system.slice", 
        "RequiresMountsFor": "/var/tmp", 
        "Restart": "no", 
        "RestartUSec": "100ms", 
        "Result": "success", 
        "RootDirectoryStartOnly": "no", 
        "RuntimeDirectoryMode": "0755", 
        "SameProcessGroup": "no", 
        "SecureBits": "0", 
        "SendSIGHUP": "no", 
        "SendSIGKILL": "yes", 
        "Slice": "system.slice", 
        "StandardError": "inherit", 
        "StandardInput": "null", 
        "StandardOutput": "journal", 
        "StartLimitAction": "none", 
        "StartLimitBurst": "5", 
        "StartLimitInterval": "10000000", 
        "StartupBlockIOWeight": "18446744073709551615", 
        "StartupCPUShares": "18446744073709551615", 
        "StatusErrno": "0", 
        "StopWhenUnneeded": "no", 
        "SubState": "dead", 
        "SyslogLevelPrefix": "yes", 
        "SyslogPriority": "30", 
        "SystemCallErrorNumber": "0", 
        "TTYReset": "no", 
        "TTYVHangup": "no", 
        "TTYVTDisallocate": "no", 
        "TasksAccounting": "no", 
        "TasksCurrent": "18446744073709551615", 
        "TasksMax": "18446744073709551615", 
        "TimeoutStartUSec": "1min 30s", 
        "TimeoutStopUSec": "1min 30s", 
        "TimerSlackNSec": "50000", 
        "Transient": "no", 
        "Type": "notify", 
        "UMask": "0022", 
        "UnitFilePreset": "disabled", 
        "UnitFileState": "disabled", 
        "WatchdogTimestampMonotonic": "0", 
        "WatchdogUSec": "0"
    }
}
[user@localhost ansible]$
...

Проверим с помощью команды curl:
...
[user@localhost ansible]$ curl 192.168.1.10
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd"><html><head>
<meta http-equiv="content-type" content="text/html; charset=UTF-8">
		<title>Apache HTTP Server Test Page powered by CentOS</title>
		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">

    <!-- Bootstrap -->
    <link href="/noindex/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="noindex/css/open-sans.css" type="text/css" />

<style type="text/css"><!--		 

body {
  font-family: "Open Sans", Helvetica, sans-serif;
  font-weight: 100;
  color: #ccc;
  background: rgba(10, 24, 55, 1);
  font-size: 16px;
}

h2, h3, h4 {
  font-weight: 200;
}

h2 {
  font-size: 28px;
}

.jumbotron {
  margin-bottom: 0;
  color: #333;
  background: rgb(212,212,221); /* Old browsers */
  background: radial-gradient(ellipse at center top, rgba(255,255,255,1) 0%,rgba(174,174,183,1) 100%); /* W3C */
}

.jumbotron h1 {
  font-size: 128px;
  font-weight: 700;
  color: white;
  text-shadow: 0px 2px 0px #abc,
               0px 4px 10px rgba(0,0,0,0.15),
               0px 5px 2px rgba(0,0,0,0.1),
               0px 6px 30px rgba(0,0,0,0.1);
}

.jumbotron p {
  font-size: 28px;
  font-weight: 100;
}

.main {
   background: white;
   color: #234;
   border-top: 1px solid rgba(0,0,0,0.12);
   padding-top: 30px;
   padding-bottom: 40px;
}

.footer {
   border-top: 1px solid rgba(255,255,255,0.2);
   padding-top: 30px;
}

    --></style>
</head>
<body>
  <div class="jumbotron text-center">
    <div class="container">
   	  <h1>Testing 123..</h1>
  		<p class="lead">This page is used to test the proper operation of the <a href="http://apache.org">Apache HTTP server</a> after it has been installed. If you can read this page it means that this site is working properly. This server is powered by <a href="http://centos.org">CentOS</a>.</p>
		</div>
  </div>
  <div class="main">
    <div class="container">
       <div class="row">
  			<div class="col-sm-6">
    			<h2>Just visiting?</h2>
			  		<p class="lead">The website you just visited is either experiencing problems or is undergoing routine maintenance.</p>
  					<p>If you would like to let the administrators of this website know that you've seen this page instead of the page you expected, you should send them e-mail. In general, mail sent to the name "webmaster" and directed to the website's domain should reach the appropriate person.</p>
  					<p>For example, if you experienced problems while visiting www.example.com, you should send e-mail to "webmaster@example.com".</p>
	  			</div>
  				<div class="col-sm-6">
	  				<h2>Are you the Administrator?</h2>
		  			<p>You should add your website content to the directory <tt>/var/www/html/</tt>.</p>
		  			<p>To prevent this page from ever being used, follow the instructions in the file <tt>/etc/httpd/conf.d/welcome.conf</tt>.</p>

	  				<h2>Promoting Apache and CentOS</h2>
			  		<p>You are free to use the images below on Apache and CentOS Linux powered HTTP servers.  Thanks for using Apache and CentOS!</p>
				  	<p><a href="http://httpd.apache.org/"><img src="images/apache_pb.gif" alt="[ Powered by Apache ]"></a> <a href="http://www.centos.org/"><img src="images/poweredby.png" alt="[ Powered by CentOS Linux ]" height="31" width="88"></a></p>
  				</div>
	  		</div>
	    </div>
		</div>
	</div>
	  <div class="footer">
      <div class="container">
        <div class="row">
          <div class="col-sm-6">          
            <h2>Important note:</h2>
            <p class="lead">The CentOS Project has nothing to do with this website or its content,
            it just provides the software that makes the website run.</p>
            
            <p>If you have issues with the content of this site, contact the owner of the domain, not the CentOS project. 
            Unless you intended to visit CentOS.org, the CentOS Project does not have anything to do with this website,
            the content or the lack of it.</p>
            <p>For example, if this website is www.example.com, you would find the owner of the example.com domain at the following WHOIS server:</p>
            <p><a href="http://www.internic.net/whois.html">http://www.internic.net/whois.html</a></p>
          </div>
          <div class="col-sm-6">
            <h2>The CentOS Project</h2>
            <p>The CentOS Linux distribution is a stable, predictable, manageable and reproduceable platform derived from 
               the sources of Red Hat Enterprise Linux (RHEL).<p>
            
            <p>Additionally to being a popular choice for web hosting, CentOS also provides a rich platform for open source communities to build upon. For more information
               please visit the <a href="http://www.centos.org/">CentOS website</a>.</p>
          </div>
        </div>
		  </div>
    </div>
  </div>
</body></html>
[user@localhost ansible]$
...
Как видим, выводит страницу Apache.

Удалим httpd:
...
[user@localhost ansible]$ ansible all -m yum -a "name=httpd state=removed" -b
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
[user@localhost ansible]$
...

Для поиска дебага, например, для "ls /home" с помощью -v, -vv, -vvv, -vvvv:
...
[user@localhost ansible]$ ansible staging_servers -m shell -a "ls -l /home"
host0 | CHANGED | rc=0 >>
total 0
drwx------. 4 vagrant vagrant 90 Jul 30 21:27 vagrant
[user@localhost ansible]$
...

...
[user@localhost ansible]$ ansible staging_servers -m shell -a "ls -l /home" -v
Using /home/user/adv/ansible/ansible.cfg as config file
host0 | CHANGED | rc=0 >>
total 0
drwx------. 4 vagrant vagrant 90 Jul 30 21:27 vagrant
[user@localhost ansible]$
...

...
[user@localhost ansible]$ ansible staging_servers -m shell -a "ls -l /home" -vv
ansible 2.9.27
  config file = /home/user/adv/ansible/ansible.cfg
  configured module search path = [u'/home/user/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Jun 28 2022, 15:30:04) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
Using /home/user/adv/ansible/ansible.cfg as config file
Skipping callback 'actionable', as we already have a stdout callback.
Skipping callback 'counter_enabled', as we already have a stdout callback.
Skipping callback 'debug', as we already have a stdout callback.
Skipping callback 'dense', as we already have a stdout callback.
Skipping callback 'dense', as we already have a stdout callback.
Skipping callback 'full_skip', as we already have a stdout callback.
Skipping callback 'json', as we already have a stdout callback.
Skipping callback 'minimal', as we already have a stdout callback.
Skipping callback 'null', as we already have a stdout callback.
Skipping callback 'oneline', as we already have a stdout callback.
Skipping callback 'selective', as we already have a stdout callback.
Skipping callback 'skippy', as we already have a stdout callback.
Skipping callback 'stderr', as we already have a stdout callback.
Skipping callback 'unixy', as we already have a stdout callback.
Skipping callback 'yaml', as we already have a stdout callback.
META: ran handlers
host0 | CHANGED | rc=0 >>
total 0
drwx------. 4 vagrant vagrant 90 Jul 30 21:27 vagrant
META: ran handlers
META: ran handlers
[user@localhost ansible]$
...

...
[user@localhost ansible]$ ansible staging_servers -m shell -a "ls -l /home" -vvv
ansible 2.9.27
  config file = /home/user/adv/ansible/ansible.cfg
  configured module search path = [u'/home/user/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Jun 28 2022, 15:30:04) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
Using /home/user/adv/ansible/ansible.cfg as config file
host_list declined parsing /home/user/adv/ansible/inventory as it did not pass its verify_file() method
script declined parsing /home/user/adv/ansible/inventory as it did not pass its verify_file() method
auto declined parsing /home/user/adv/ansible/inventory as it did not pass its verify_file() method
Parsed /home/user/adv/ansible/inventory inventory source with ini plugin
Skipping callback 'actionable', as we already have a stdout callback.
Skipping callback 'counter_enabled', as we already have a stdout callback.
Skipping callback 'debug', as we already have a stdout callback.
Skipping callback 'dense', as we already have a stdout callback.
Skipping callback 'dense', as we already have a stdout callback.
Skipping callback 'full_skip', as we already have a stdout callback.
Skipping callback 'json', as we already have a stdout callback.
Skipping callback 'minimal', as we already have a stdout callback.
Skipping callback 'null', as we already have a stdout callback.
Skipping callback 'oneline', as we already have a stdout callback.
Skipping callback 'selective', as we already have a stdout callback.
Skipping callback 'skippy', as we already have a stdout callback.
Skipping callback 'stderr', as we already have a stdout callback.
Skipping callback 'unixy', as we already have a stdout callback.
Skipping callback 'yaml', as we already have a stdout callback.
META: ran handlers
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'echo ~vagrant && sleep 0'"'"''
<192.168.1.10> (0, '/home/vagrant\n', '')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'( umask 77 && mkdir -p "` echo /home/vagrant/.ansible/tmp `"&& mkdir "` echo /home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472 `" && echo ansible-tmp-1659219681.54-26302-190980648517472="` echo /home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472 `" ) && sleep 0'"'"''
<192.168.1.10> (0, 'ansible-tmp-1659219681.54-26302-190980648517472=/home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472\n', '')
<host0> Attempting python interpreter discovery
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'echo PLATFORM; uname; echo FOUND; command -v '"'"'"'"'"'"'"'"'/usr/bin/python'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python3.7'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python3.6'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python3.5'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python2.7'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python2.6'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'/usr/libexec/platform-python'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'/usr/bin/python3'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python'"'"'"'"'"'"'"'"'; echo ENDFOUND && sleep 0'"'"''
<192.168.1.10> (0, 'PLATFORM\nLinux\nFOUND\n/usr/bin/python\n/usr/bin/python2.7\n/usr/libexec/platform-python\n/usr/bin/python\nENDFOUND\n', '')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'/usr/bin/python && sleep 0'"'"''
<192.168.1.10> (0, '{"osrelease_content": "NAME=\\"CentOS Linux\\"\\nVERSION=\\"7 (Core)\\"\\nID=\\"centos\\"\\nID_LIKE=\\"rhel fedora\\"\\nVERSION_ID=\\"7\\"\\nPRETTY_NAME=\\"CentOS Linux 7 (Core)\\"\\nANSI_COLOR=\\"0;31\\"\\nCPE_NAME=\\"cpe:/o:centos:centos:7\\"\\nHOME_URL=\\"https://www.centos.org/\\"\\nBUG_REPORT_URL=\\"https://bugs.centos.org/\\"\\n\\nCENTOS_MANTISBT_PROJECT=\\"CentOS-7\\"\\nCENTOS_MANTISBT_PROJECT_VERSION=\\"7\\"\\nREDHAT_SUPPORT_PRODUCT=\\"centos\\"\\nREDHAT_SUPPORT_PRODUCT_VERSION=\\"7\\"\\n\\n", "platform_dist_result": ["centos", "7.8.2003", "Core"]}\n', '')
Using module file /usr/lib/python2.7/site-packages/ansible/modules/commands/command.py
<192.168.1.10> PUT /home/user/.ansible/tmp/ansible-local-26294SkDIna/tmp1xwNsM TO /home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472/AnsiballZ_command.py
<192.168.1.10> SSH: EXEC sftp -b - -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 '[192.168.1.10]'
<192.168.1.10> (0, 'sftp> put /home/user/.ansible/tmp/ansible-local-26294SkDIna/tmp1xwNsM /home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472/AnsiballZ_command.py\n', '')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'chmod u+x /home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472/ /home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472/AnsiballZ_command.py && sleep 0'"'"''
<192.168.1.10> (0, '', '')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 -tt 192.168.1.10 '/bin/sh -c '"'"'/usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472/AnsiballZ_command.py && sleep 0'"'"''
<192.168.1.10> (0, '\r\n{"changed": true, "end": "2022-07-30 22:21:19.054081", "stdout": "total 0\\ndrwx------. 4 vagrant vagrant 90 Jul 30 21:27 vagrant", "cmd": "ls -l /home", "rc": 0, "start": "2022-07-30 22:21:19.047599", "stderr": "", "delta": "0:00:00.006482", "invocation": {"module_args": {"creates": null, "executable": null, "_uses_shell": true, "strip_empty_ends": true, "_raw_params": "ls -l /home", "removes": null, "argv": null, "warn": true, "chdir": null, "stdin_add_newline": true, "stdin": null}}}\r\n', 'Shared connection to 192.168.1.10 closed.\r\n')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'rm -f -r /home/vagrant/.ansible/tmp/ansible-tmp-1659219681.54-26302-190980648517472/ > /dev/null 2>&1 && sleep 0'"'"''
<192.168.1.10> (0, '', '')
host0 | CHANGED | rc=0 >>
total 0
drwx------. 4 vagrant vagrant 90 Jul 30 21:27 vagrant
META: ran handlers
META: ran handlers
[user@localhost ansible]$
...

...
[user@localhost ansible]$ ansible staging_servers -m shell -a "ls -l /home" -vvvv
ansible 2.9.27
  config file = /home/user/adv/ansible/ansible.cfg
  configured module search path = [u'/home/user/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Jun 28 2022, 15:30:04) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
Using /home/user/adv/ansible/ansible.cfg as config file
setting up inventory plugins
host_list declined parsing /home/user/adv/ansible/inventory as it did not pass its verify_file() method
script declined parsing /home/user/adv/ansible/inventory as it did not pass its verify_file() method
auto declined parsing /home/user/adv/ansible/inventory as it did not pass its verify_file() method
Parsed /home/user/adv/ansible/inventory inventory source with ini plugin
Loading callback plugin minimal of type stdout, v2.0 from /usr/lib/python2.7/site-packages/ansible/plugins/callback/minimal.pyc
Skipping callback 'actionable', as we already have a stdout callback.
Skipping callback 'counter_enabled', as we already have a stdout callback.
Skipping callback 'debug', as we already have a stdout callback.
Skipping callback 'dense', as we already have a stdout callback.
Skipping callback 'dense', as we already have a stdout callback.
Skipping callback 'full_skip', as we already have a stdout callback.
Skipping callback 'json', as we already have a stdout callback.
Skipping callback 'minimal', as we already have a stdout callback.
Skipping callback 'null', as we already have a stdout callback.
Skipping callback 'oneline', as we already have a stdout callback.
Skipping callback 'selective', as we already have a stdout callback.
Skipping callback 'skippy', as we already have a stdout callback.
Skipping callback 'stderr', as we already have a stdout callback.
Skipping callback 'unixy', as we already have a stdout callback.
Skipping callback 'yaml', as we already have a stdout callback.
META: ran handlers
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'echo ~vagrant && sleep 0'"'"''
<192.168.1.10> (0, '/home/vagrant\n', 'OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017\r\ndebug1: Reading configuration data /etc/ssh/ssh_config\r\ndebug1: /etc/ssh/ssh_config line 58: Applying options for *\r\ndebug1: auto-mux: Trying existing master\r\ndebug2: fd 3 setting O_NONBLOCK\r\ndebug2: mux_client_hello_exchange: master version 4\r\ndebug3: mux_client_forwards: request forwardings: 0 local, 0 remote\r\ndebug3: mux_client_request_session: entering\r\ndebug3: mux_client_request_alive: entering\r\ndebug3: mux_client_request_alive: done pid = 26222\r\ndebug3: mux_client_request_session: session request sent\r\ndebug1: mux_client_request_session: master session id: 2\r\ndebug3: mux_client_read_packet: read header failed: Broken pipe\r\ndebug2: Received exit status from master 0\r\n')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'( umask 77 && mkdir -p "` echo /home/vagrant/.ansible/tmp `"&& mkdir "` echo /home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308 `" && echo ansible-tmp-1659219724.25-26335-48953814634308="` echo /home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308 `" ) && sleep 0'"'"''
<192.168.1.10> (0, 'ansible-tmp-1659219724.25-26335-48953814634308=/home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308\n', 'OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017\r\ndebug1: Reading configuration data /etc/ssh/ssh_config\r\ndebug1: /etc/ssh/ssh_config line 58: Applying options for *\r\ndebug1: auto-mux: Trying existing master\r\ndebug2: fd 3 setting O_NONBLOCK\r\ndebug2: mux_client_hello_exchange: master version 4\r\ndebug3: mux_client_forwards: request forwardings: 0 local, 0 remote\r\ndebug3: mux_client_request_session: entering\r\ndebug3: mux_client_request_alive: entering\r\ndebug3: mux_client_request_alive: done pid = 26222\r\ndebug3: mux_client_request_session: session request sent\r\ndebug1: mux_client_request_session: master session id: 2\r\ndebug3: mux_client_read_packet: read header failed: Broken pipe\r\ndebug2: Received exit status from master 0\r\n')
<host0> Attempting python interpreter discovery
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'echo PLATFORM; uname; echo FOUND; command -v '"'"'"'"'"'"'"'"'/usr/bin/python'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python3.7'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python3.6'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python3.5'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python2.7'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python2.6'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'/usr/libexec/platform-python'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'/usr/bin/python3'"'"'"'"'"'"'"'"'; command -v '"'"'"'"'"'"'"'"'python'"'"'"'"'"'"'"'"'; echo ENDFOUND && sleep 0'"'"''
<192.168.1.10> (0, 'PLATFORM\nLinux\nFOUND\n/usr/bin/python\n/usr/bin/python2.7\n/usr/libexec/platform-python\n/usr/bin/python\nENDFOUND\n', 'OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017\r\ndebug1: Reading configuration data /etc/ssh/ssh_config\r\ndebug1: /etc/ssh/ssh_config line 58: Applying options for *\r\ndebug1: auto-mux: Trying existing master\r\ndebug2: fd 3 setting O_NONBLOCK\r\ndebug2: mux_client_hello_exchange: master version 4\r\ndebug3: mux_client_forwards: request forwardings: 0 local, 0 remote\r\ndebug3: mux_client_request_session: entering\r\ndebug3: mux_client_request_alive: entering\r\ndebug3: mux_client_request_alive: done pid = 26222\r\ndebug3: mux_client_request_session: session request sent\r\ndebug1: mux_client_request_session: master session id: 2\r\ndebug3: mux_client_read_packet: read header failed: Broken pipe\r\ndebug2: Received exit status from master 0\r\n')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'/usr/bin/python && sleep 0'"'"''
<192.168.1.10> (0, '{"osrelease_content": "NAME=\\"CentOS Linux\\"\\nVERSION=\\"7 (Core)\\"\\nID=\\"centos\\"\\nID_LIKE=\\"rhel fedora\\"\\nVERSION_ID=\\"7\\"\\nPRETTY_NAME=\\"CentOS Linux 7 (Core)\\"\\nANSI_COLOR=\\"0;31\\"\\nCPE_NAME=\\"cpe:/o:centos:centos:7\\"\\nHOME_URL=\\"https://www.centos.org/\\"\\nBUG_REPORT_URL=\\"https://bugs.centos.org/\\"\\n\\nCENTOS_MANTISBT_PROJECT=\\"CentOS-7\\"\\nCENTOS_MANTISBT_PROJECT_VERSION=\\"7\\"\\nREDHAT_SUPPORT_PRODUCT=\\"centos\\"\\nREDHAT_SUPPORT_PRODUCT_VERSION=\\"7\\"\\n\\n", "platform_dist_result": ["centos", "7.8.2003", "Core"]}\n', 'OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017\r\ndebug1: Reading configuration data /etc/ssh/ssh_config\r\ndebug1: /etc/ssh/ssh_config line 58: Applying options for *\r\ndebug1: auto-mux: Trying existing master\r\ndebug2: fd 3 setting O_NONBLOCK\r\ndebug2: mux_client_hello_exchange: master version 4\r\ndebug3: mux_client_forwards: request forwardings: 0 local, 0 remote\r\ndebug3: mux_client_request_session: entering\r\ndebug3: mux_client_request_alive: entering\r\ndebug3: mux_client_request_alive: done pid = 26222\r\ndebug3: mux_client_request_session: session request sent\r\ndebug1: mux_client_request_session: master session id: 2\r\ndebug3: mux_client_read_packet: read header failed: Broken pipe\r\ndebug2: Received exit status from master 0\r\n')
Using module file /usr/lib/python2.7/site-packages/ansible/modules/commands/command.py
<192.168.1.10> PUT /home/user/.ansible/tmp/ansible-local-26327J8O4Cc/tmpgGzi98 TO /home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308/AnsiballZ_command.py
<192.168.1.10> SSH: EXEC sftp -b - -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 '[192.168.1.10]'
<192.168.1.10> (0, 'sftp> put /home/user/.ansible/tmp/ansible-local-26327J8O4Cc/tmpgGzi98 /home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308/AnsiballZ_command.py\n', 'OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017\r\ndebug1: Reading configuration data /etc/ssh/ssh_config\r\ndebug1: /etc/ssh/ssh_config line 58: Applying options for *\r\ndebug1: auto-mux: Trying existing master\r\ndebug2: fd 3 setting O_NONBLOCK\r\ndebug2: mux_client_hello_exchange: master version 4\r\ndebug3: mux_client_forwards: request forwardings: 0 local, 0 remote\r\ndebug3: mux_client_request_session: entering\r\ndebug3: mux_client_request_alive: entering\r\ndebug3: mux_client_request_alive: done pid = 26222\r\ndebug3: mux_client_request_session: session request sent\r\ndebug1: mux_client_request_session: master session id: 2\r\ndebug2: Remote version: 3\r\ndebug2: Server supports extension "posix-rename@openssh.com" revision 1\r\ndebug2: Server supports extension "statvfs@openssh.com" revision 2\r\ndebug2: Server supports extension "fstatvfs@openssh.com" revision 2\r\ndebug2: Server supports extension "hardlink@openssh.com" revision 1\r\ndebug2: Server supports extension "fsync@openssh.com" revision 1\r\ndebug3: Sent message fd 5 T:16 I:1\r\ndebug3: SSH_FXP_REALPATH . -> /home/vagrant size 0\r\ndebug3: Looking up /home/user/.ansible/tmp/ansible-local-26327J8O4Cc/tmpgGzi98\r\ndebug3: Sent message fd 5 T:17 I:2\r\ndebug3: Received stat reply T:101 I:2\r\ndebug1: Couldn\'t stat remote file: No such file or directory\r\ndebug3: Sent message SSH2_FXP_OPEN I:3 P:/home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308/AnsiballZ_command.py\r\ndebug3: Sent message SSH2_FXP_WRITE I:4 O:0 S:32768\r\ndebug3: SSH2_FXP_STATUS 0\r\ndebug3: In write loop, ack for 4 32768 bytes at 0\r\ndebug3: Sent message SSH2_FXP_WRITE I:5 O:32768 S:32768\r\ndebug3: Sent message SSH2_FXP_WRITE I:6 O:65536 S:32768\r\ndebug3: Sent message SSH2_FXP_WRITE I:7 O:98304 S:20260\r\ndebug3: SSH2_FXP_STATUS 0\r\ndebug3: In write loop, ack for 5 32768 bytes at 32768\r\ndebug3: SSH2_FXP_STATUS 0\r\ndebug3: In write loop, ack for 6 32768 bytes at 65536\r\ndebug3: SSH2_FXP_STATUS 0\r\ndebug3: In write loop, ack for 7 20260 bytes at 98304\r\ndebug3: Sent message SSH2_FXP_CLOSE I:4\r\ndebug3: SSH2_FXP_STATUS 0\r\ndebug3: mux_client_read_packet: read header failed: Broken pipe\r\ndebug2: Received exit status from master 0\r\n')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'chmod u+x /home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308/ /home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308/AnsiballZ_command.py && sleep 0'"'"''
<192.168.1.10> (0, '', 'OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017\r\ndebug1: Reading configuration data /etc/ssh/ssh_config\r\ndebug1: /etc/ssh/ssh_config line 58: Applying options for *\r\ndebug1: auto-mux: Trying existing master\r\ndebug2: fd 3 setting O_NONBLOCK\r\ndebug2: mux_client_hello_exchange: master version 4\r\ndebug3: mux_client_forwards: request forwardings: 0 local, 0 remote\r\ndebug3: mux_client_request_session: entering\r\ndebug3: mux_client_request_alive: entering\r\ndebug3: mux_client_request_alive: done pid = 26222\r\ndebug3: mux_client_request_session: session request sent\r\ndebug1: mux_client_request_session: master session id: 2\r\ndebug3: mux_client_read_packet: read header failed: Broken pipe\r\ndebug2: Received exit status from master 0\r\n')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 -tt 192.168.1.10 '/bin/sh -c '"'"'/usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308/AnsiballZ_command.py && sleep 0'"'"''
<192.168.1.10> (0, '\r\n{"changed": true, "end": "2022-07-30 22:22:01.772699", "stdout": "total 0\\ndrwx------. 4 vagrant vagrant 90 Jul 30 21:27 vagrant", "cmd": "ls -l /home", "rc": 0, "start": "2022-07-30 22:22:01.766471", "stderr": "", "delta": "0:00:00.006228", "invocation": {"module_args": {"creates": null, "executable": null, "_uses_shell": true, "strip_empty_ends": true, "_raw_params": "ls -l /home", "removes": null, "argv": null, "warn": true, "chdir": null, "stdin_add_newline": true, "stdin": null}}}\r\n', 'OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017\r\ndebug1: Reading configuration data /etc/ssh/ssh_config\r\ndebug1: /etc/ssh/ssh_config line 58: Applying options for *\r\ndebug1: auto-mux: Trying existing master\r\ndebug2: fd 3 setting O_NONBLOCK\r\ndebug2: mux_client_hello_exchange: master version 4\r\ndebug3: mux_client_forwards: request forwardings: 0 local, 0 remote\r\ndebug3: mux_client_request_session: entering\r\ndebug3: mux_client_request_alive: entering\r\ndebug3: mux_client_request_alive: done pid = 26222\r\ndebug3: mux_client_request_session: session request sent\r\ndebug1: mux_client_request_session: master session id: 2\r\ndebug3: mux_client_read_packet: read header failed: Broken pipe\r\ndebug2: Received exit status from master 0\r\nShared connection to 192.168.1.10 closed.\r\n')
<192.168.1.10> ESTABLISH SSH CONNECTION FOR USER: vagrant
<192.168.1.10> SSH: EXEC ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no -o Port=22 -o 'IdentityFile="/home/user/.ssh/staging"' -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o 'User="vagrant"' -o ConnectTimeout=10 -o ControlPath=/home/user/.ansible/cp/bb5d415b32 192.168.1.10 '/bin/sh -c '"'"'rm -f -r /home/vagrant/.ansible/tmp/ansible-tmp-1659219724.25-26335-48953814634308/ > /dev/null 2>&1 && sleep 0'"'"''
<192.168.1.10> (0, '', 'OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017\r\ndebug1: Reading configuration data /etc/ssh/ssh_config\r\ndebug1: /etc/ssh/ssh_config line 58: Applying options for *\r\ndebug1: auto-mux: Trying existing master\r\ndebug2: fd 3 setting O_NONBLOCK\r\ndebug2: mux_client_hello_exchange: master version 4\r\ndebug3: mux_client_forwards: request forwardings: 0 local, 0 remote\r\ndebug3: mux_client_request_session: entering\r\ndebug3: mux_client_request_alive: entering\r\ndebug3: mux_client_request_alive: done pid = 26222\r\ndebug3: mux_client_request_session: session request sent\r\ndebug1: mux_client_request_session: master session id: 2\r\ndebug3: mux_client_read_packet: read header failed: Broken pipe\r\ndebug2: Received exit status from master 0\r\n')
host0 | CHANGED | rc=0 >>
total 0
drwx------. 4 vagrant vagrant 90 Jul 30 21:27 vagrant
META: ran handlers
META: ran handlers
[user@localhost ansible]$
...

То есть чем больше "v", тем более подробную информацию видим, что происходит после запуска команды.

Просмотреть все модули ansible:
...
[user@localhost ansible]$ ansible-doc -l
...

Лучше всего документацию искать в браузере, набрав в пооисковой строке, например, ansible file, ansible yum, ansible service  и т. д.

### 08-Ansible - Правила Формата YAML

YAML-файлы составляются следующим образом, начиная с трёх тире "---":
...
---
 - fruits:
       - apple
       - orange
       - mango

 - vegetables:
       - carrots
       - cucumbers


 - vasya:
       nick: vasek
       position: developer
       skills:
           - python
           - perl
           - php

 - petya:
       nick: "pet: krutoy"
       position: manager
       skills:
           - manage
           - make_noise
...

Можно ещё так:
...
---
 - fruits: ['apple', 'orange', 'mango']

 - vegetables: ['carrots', 'cucumbers']

 - vasya: { nick: vasek, position: developer, skills: ['python', 'perl', 'php'] }

 - petya: { nick: "pet: krutoy", position: manager, skills: ['manage', 'make_noise'] }
...

Если нет перечисления, то тире "-" в начале строки не ставится.

### 09-Ansible - Перенос переменных в group_vars

Переносим переменные из файла inventory в отдельные файлы с названием группы серверов.
Inventory файл выглядит так:
...
[user@localhost ansible]$ cat inventory
[staging_servers]
host0 ansible_host=192.168.1.10 ansible_port=22 ansible_user=vagrant ansible_private_key_file=/home/user/.ssh/staging

[prod_servers]
host1 ansible_host=192.168.1.11
host2 ansible_host=192.168.1.12

[prod_servers:vars]
ansible_port=22
ansible_user=vagrant
ansible_private_key_file=/home/user/.ssh/product

[user@localhost ansible]$
...

Создадим директорию group_vars:
...
[user@localhost ansible]$ mkdir group_vars
[user@localhost ansible]$
...

В этой директории создадим файл prod_servers:
...
[user@localhost ansible]$ vi ./group_vars/prod_servers
...

...
---
ansible_port : 22
ansible_user : vagrant
ansible_private_key_file : /home/user/.ssh/product
...

Вместо равно "=" ставим двоеточие ":", количество пробелов сколько угодно.
В inventory файле убираем секцию [prod_servers:vars], теперь inventory файл выглядит так:
...
[user@localhost ansible]$ cat ./inventory
[staging_servers]
host0 ansible_host=192.168.1.10 ansible_port=22 ansible_user=vagrant ansible_private_key_file=/home/user/.ssh/staging

[prod_servers]
host1 ansible_host=192.168.1.11
host2 ansible_host=192.168.1.12

[user@localhost ansible]$
...

Проверим с помощью следующей команды:
...
[user@localhost ansible]$ ansible-inventory --list
{
    "_meta": {
        "hostvars": {
            "host0": {
                "ansible_host": "192.168.1.10", 
                "ansible_port": 22, 
                "ansible_private_key_file": "/home/user/.ssh/staging", 
                "ansible_user": "vagrant"
            }, 
            "host1": {
                "ansible_host": "192.168.1.11", 
                "ansible_port": 22, 
                "ansible_private_key_file": "/home/user/.ssh/product", 
                "ansible_user": "vagrant"
            }, 
            "host2": {
                "ansible_host": "192.168.1.12", 
                "ansible_port": 22, 
                "ansible_private_key_file": "/home/user/.ssh/product", 
                "ansible_user": "vagrant"
            }
        }
    }, 
    "all": {
        "children": [
            "prod_servers", 
            "staging_servers", 
            "ungrouped"
        ]
    }, 
    "prod_servers": {
        "hosts": [
            "host1", 
            "host2"
        ]
    }, 
    "staging_servers": {
        "hosts": [
            "host0"
        ]
    }
}
[user@localhost ansible]$
...
Как видим, изменений в списке inventory нет, всё то же самое.

### 10-Ansible - Первые Playbook

У нас есть конфиг файл ansible.cfg:
...
[user@localhost ansible]$ cat ./ansible.cfg 
[defaults]
host_key_checking = false
inventory         = ./inventory

[user@localhost ansible]$
...

Есть inventory файл inventory:
...
[user@localhost ansible]$ cat ./inventory
[staging_servers]
host0 ansible_host=192.168.1.10 ansible_port=22 ansible_user=vagrant ansible_private_key_file=/home/user/.ssh/staging

[prod_servers]
host1 ansible_host=192.168.1.11
host2 ansible_host=192.168.1.12

[user@localhost ansible]$
...

Есть файл, содержащий переменные группы prod_servers (файлов переменных может быть несколько):
...
[user@localhost ansible]$ cat ./group_vars/prod_servers 
---
ansible_port : 22
ansible_user : vagrant
ansible_private_key_file : /home/user/.ssh/product
[user@localhost ansible]$
...

Создадим наш первый playbook файл по пингованию наших серверов:
...
[user@localhost ansible]$ vi playbook1.yml
...

...
---
- name: Test Connection to my servers  # Название нашего Playbook
  hosts: all                           # все сервера
  become: yes                          # аналогично sudo

  tasks:

  - name: Ping my servers              # имя команды
    ping:                              # сам модуль
...

Запустим с помощью команды ansible-playbook:
...
[user@localhost ansible]$ ansible-playbook ./playbook1.yml 

PLAY [Test Connection to my servers] **************************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Ping my servers] ****************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

PLAY RECAP ****************************************************************************
host0                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

Создадим playbook файл по установке и запуске Apache:
...
[user@localhost ansible]$ vi playbook2.yml
...

...
---
- name: Install default Apache Web Server
  hosts: all
  become: yes

  tasks:
  - name: Install Apache Web Server
    yum: name=httpd state=latest

  - name: Start Apache service and Enable it on the every boot
    service: name=httpd state=started enabled=yes
...

Запустим с playbook2.yml:
...
[user@localhost ansible]$ ansible-playbook ./playbook2.yml 

PLAY [Install default Apache Web Server] **********************************************

TASK [Gathering Facts] ****************************************************************
ok: [host2]
ok: [host0]
ok: [host1]

TASK [Install Apache Web Server] ******************************************************
changed: [host2]
changed: [host0]
changed: [host1]

TASK [Start Apache service and Enable it on the every boot] ***************************
changed: [host0]
changed: [host2]
changed: [host1]

PLAY RECAP ****************************************************************************
host0                      : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

На всех серверах успешно установились и запустились Apache.
в этом можно убедиться с помощью команды curl 192.168.1.10 или в браузере, что выдаст тестовую страницу Apache.

попробуем повторить команду:
...
[user@localhost ansible]$ ansible-playbook ./playbook2.yml 

PLAY [Install default Apache Web Server] **********************************************

TASK [Gathering Facts] ****************************************************************
ok: [host0]
ok: [host2]
ok: [host1]

TASK [Install Apache Web Server] ******************************************************
ok: [host0]
ok: [host2]
ok: [host1]

TASK [Start Apache service and Enable it on the every boot] ***************************
ok: [host2]
ok: [host0]
ok: [host1]

PLAY RECAP ****************************************************************************
host0                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...
Покажет, что всё установлено и запущено, никаких изменений не произошло.

Удалим Apache с помощью обычной ad-hoc команды:
...
[user@localhost ansible]$ ansible all -m yum -a "name=httpd state=absent" -b
host0 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
host2 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
host1 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    }, 
    "changed": true, 
    "changes": {
        "removed": [
            "httpd"
        ]
    }, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package httpd.x86_64 0:2.4.6-97.el7.centos.5 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package      Arch          Version                       Repository       Size\n================================================================================\nRemoving:\n httpd        x86_64        2.4.6-97.el7.centos.5         @updates        9.4 M\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 9.4 M\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n  Verifying  : httpd-2.4.6-97.el7.centos.5.x86_64                           1/1 \n\nRemoved:\n  httpd.x86_64 0:2.4.6-97.el7.centos.5                                          \n\nComplete!\n"
    ]
}
[user@localhost ansible]$
...

Для следующего примера создадим html-страницу. 
Создадим каталог mywebsite:
...
[user@localhost ansible]$ mkdir ./MyWebSite
[user@localhost ansible]$
...

В этой директории создадим html страницу index.html:
...
[user@localhost ansible]$ vi ./MyWebSite/index.html
...

...
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8,10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style.letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout('Elastic()', 108)
}
</SCRIPT>
<BODY bgcolor="black" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="green"><h2>This WebServer Build by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
</BODY>
</HTML>
...

Создадим следующий playbook файл:
...
[user@localhost ansible]$ vi ./playbook3.yml
...

...
---
- name: Install Apache and Upload my Web Page
  hosts: all
  become: yes

  vars:
    source_file: ./MyWebSite/index.html
    destin_file: /var/www/html

  tasks:
  - name: Install Apache Web Server
    yum: name=httpd state=latest

  - name: Copy My Web Page to Servers
    copy: src={{ source_file }} dest={{ destin_file }} mode=0666
    notify: Restart Apache   # Вызовет handlers с именем Restart Apache

  - name: Start Apache service and Enable it on the every boot
    service: name=httpd state=started enabled=yes


  handlers:
  - name: Restart Apache
    service: name=httpd state=restarted
...

Запускаем с playbook3.yml:
...
[user@localhost ansible]$ ansible-playbook ./playbook3.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Install Apache Web Server] ******************************************************
changed: [host0]
changed: [host2]
changed: [host1]

TASK [Copy My Web Page to Servers] ****************************************************
changed: [host2]
changed: [host1]
changed: [host0]

TASK [Start Apache service and Enable it on the every boot] ***************************
changed: [host2]
changed: [host1]
changed: [host0]

RUNNING HANDLER [Restart Apache] ******************************************************
changed: [host2]
changed: [host1]
changed: [host0]

PLAY RECAP ****************************************************************************
host0                      : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

Apache установлено и запущено, наша веб-страница загружена на сервера. Убедимся с помощью команды curl:
...
[user@localhost ansible]$ curl 192.168.1.10
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8,10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style.letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout('Elastic()', 108)
}
</SCRIPT>
<BODY bgcolor="black" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="green"><h2>This WebServer Build by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
</BODY>
</HTML>
[user@localhost ansible]$
...
Как видим, отображается наша веб-страница index.html.

Повторим команду установки Apache:
...
[user@localhost ansible]$ ansible-playbook ./playbook3.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host0]
ok: [host2]
ok: [host1]

TASK [Install Apache Web Server] ******************************************************
ok: [host2]
ok: [host1]
ok: [host0]

TASK [Copy My Web Page to Servers] ****************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Start Apache service and Enable it on the every boot] ***************************
ok: [host0]
ok: [host2]
ok: [host1]

PLAY RECAP ****************************************************************************
host0                      : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...
Как видим, изменений не произошло.

Изменим что-нибудь в нашем index.html, например, цвет фона с чёрного на серый:
...
<BODY bgcolor="gray" onLoad=Elastic()>
...

Запустим ещё раз команду:
...
[user@localhost ansible]$ ansible-playbook ./playbook3.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

TASK [Install Apache Web Server] ******************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Copy My Web Page to Servers] ****************************************************
changed: [host0]
changed: [host1]
changed: [host2]

TASK [Start Apache service and Enable it on the every boot] ***************************
ok: [host2]
ok: [host1]
ok: [host0]

RUNNING HANDLER [Restart Apache] ******************************************************
changed: [host0]
changed: [host2]
changed: [host1]

PLAY RECAP ****************************************************************************
host0                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

Как видим, произошло два изменения:
- скопировался на все сервера измененный файл index.html;
- перезапустился Apache (сработал handler "Restart Apache").

### 11-Ansible - Переменные - Debug, Set_fact, Register

Создадим новый playbook файл:
...
[user@localhost ansible]$ vi ./playbook4.yml
...

...
---
- name: My Super Puper Playbook for Variables Lesson
  hosts: all
  become: yes

  vars:
    message1: Hello
    message2: World
    secret  : HKDN97D6HDN532B

  tasks:

  - name: Print Secret variable
    debug:
      var: secret

  - debug:
      msg: "Sekretnoe slovo: {{ secret }}"
...

Запустим:
...
[user@localhost ansible]$ ansible-playbook ./playbook4.yml 

PLAY [My Super Puper Playbook for Variables Lesson] ***********************************

TASK [Gathering Facts] ****************************************************************
ok: [host2]
ok: [host1]
ok: [host0]

TASK [Print Secret variable] **********************************************************
ok: [host0] => {
    "secret": "HKDN97D6HDN532B" <--Вывел переменную secret
}
ok: [host1] => {
    "secret": "HKDN97D6HDN532B" <--Вывел переменную secret
}
ok: [host2] => {
    "secret": "HKDN97D6HDN532B" <--Вывел переменную secret
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B" <--Вывел переменную secret
}
ok: [host1] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B" <--Вывел переменную secret
}
ok: [host2] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B" <--Вывел переменную secret
}

PLAY RECAP ****************************************************************************
host0                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

В следующем примере в inventory файл внесём изменения, для каждого сервера переменную owner:
...
[user@localhost ansible]$ vi ./inventory
...

...
[staging_servers]
host0 ansible_host=192.168.1.10 ansible_port=22 ansible_user=vagrant ansible_private_key_file=/home/user/.ssh/staging owner=Ivan

[prod_servers]
host1 ansible_host=192.168.1.11 owner=Petya
host2 ansible_host=192.168.1.12 owner=Vasya
...

Также внесём изменение в конец нашего playbook файла строки:
...
  - debug:
      msg: "Vladeletc Etogo Servera -->{{ owner }}<--"
...

то есть получится следующим образом:
...
[user@localhost ansible]$ vi ./playbook4.yml
...

...
---
- name: My Super Puper Playbook for Variables Lesson
  hosts: all
  become: yes

  vars:
    message1: Hello
    message2: World
    secret  : HKDN97D6HDN532B

  tasks:

  - name: Print Secret variable
    debug:
      var: secret

  - debug:
      msg: "Sekretnoe slovo: {{ secret }}"

  - debug:
      msg: "Vladeletc Etogo Servera -->{{ owner }}<--"
...

Запустим:
...
[user@localhost ansible]$ ansible-playbook ./playbook4.yml 

PLAY [My Super Puper Playbook for Variables Lesson] ***********************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host2]
ok: [host0]

TASK [Print Secret variable] **********************************************************
ok: [host0] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host1] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host2] => {
    "secret": "HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host1] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host2] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Vladeletc Etogo Servera -->Ivan<--" <--Вывел переменную owner сервера host0
}
ok: [host1] => {
    "msg": "Vladeletc Etogo Servera -->Petya<--" <--Вывел переменную owner сервера host1
}
ok: [host2] => {
    "msg": "Vladeletc Etogo Servera -->Vasya<--" <--Вывел переменную owner сервера host2
}

PLAY RECAP ****************************************************************************
host0                      : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

Внесём ещё изменеия в playbook файл, добавив строки:
...
  - set_fact: full_message="{{ message1 }} {{ message2 }} from {{ owner }}"

  - debug:
      var: full_message
...

в итоге получим:
...
---
- name: My Super Puper Playbook for Variables Lesson
  hosts: all
  become: yes

  vars:
    message1: Hello
    message2: World
    secret  : HKDN97D6HDN532B

  tasks:

  - name: Print Secret variable
    debug:
      var: secret

  - debug:
      msg: "Sekretnoe slovo: {{ secret }}"

  - debug:
      msg: "Vladeletc Etogo Servera -->{{ owner }}<--"

  - set_fact: full_message="{{ message1 }} {{ message2 }} from {{ owner }}"

  - debug:
      var: full_message
...

Запустим:
...
[user@localhost ansible]$ ansible-playbook ./playbook4.yml 

PLAY [My Super Puper Playbook for Variables Lesson] ***********************************

TASK [Gathering Facts] ****************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

TASK [Print Secret variable] **********************************************************
ok: [host0] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host1] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host2] => {
    "secret": "HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host1] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host2] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Vladeletc Etogo Servera -->Ivan<--"
}
ok: [host1] => {
    "msg": "Vladeletc Etogo Servera -->Petya<--"
}
ok: [host2] => {
    "msg": "Vladeletc Etogo Servera -->Vasya<--"
}

TASK [set_fact] ***********************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

TASK [debug] **************************************************************************
ok: [host0] => {
    "full_message": "Hello World from Ivan" <--Вывело переменную full_message для host0
}
ok: [host1] => {
    "full_message": "Hello World from Petya" <--Вывело переменную full_message для host1
}
ok: [host2] => {
    "full_message": "Hello World from Vasya" <--Вывело переменную full_message для host2
}

PLAY RECAP ****************************************************************************
host0                      : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

Для следующего примера найдём переменную ansible_distribution для всех серверов с помощью следующей команды:
...
[user@localhost ansible]$ ansible all -m setup | grep ansible_distribution
        "ansible_distribution": "CentOS",         <-----Искомая переменная
        "ansible_distribution_file_parsed": true, 
        "ansible_distribution_file_path": "/etc/redhat-release", 
        "ansible_distribution_file_variety": "RedHat", 
        "ansible_distribution_major_version": "7", 
        "ansible_distribution_release": "Core", 
        "ansible_distribution_version": "7.8", 
        "ansible_distribution": "CentOS",         <-----Искомая переменная
        "ansible_distribution_file_parsed": true, 
        "ansible_distribution_file_path": "/etc/redhat-release", 
        "ansible_distribution_file_variety": "RedHat", 
        "ansible_distribution_major_version": "7", 
        "ansible_distribution_release": "Core", 
        "ansible_distribution_version": "7.8", 
        "ansible_distribution": "CentOS",         <-----Искомая переменная
        "ansible_distribution_file_parsed": true, 
        "ansible_distribution_file_path": "/etc/redhat-release", 
        "ansible_distribution_file_variety": "RedHat", 
        "ansible_distribution_major_version": "7", 
        "ansible_distribution_release": "Core", 
        "ansible_distribution_version": "7.8"
...

В playbook файл добавим строки для отображения переменной ansible_distribution:
...
  - debug:
      var: ansible_distribution
...

В итоге:
...
---
- name: My Super Puper Playbook for Variables Lesson
  hosts: all
  become: yes

  vars:
    message1: Hello
    message2: World
    secret  : HKDN97D6HDN532B

  tasks:

  - name: Print Secret variable
    debug:
      var: secret

  - debug:
      msg: "Sekretnoe slovo: {{ secret }}"

  - debug:
      msg: "Vladeletc Etogo Servera -->{{ owner }}<--"

  - set_fact: full_message="{{ message1 }} {{ message2 }} from {{ owner }}"

  - debug:
      var: full_message

  - debug:
      var: ansible_distribution
...

Запускаем:
...
[user@localhost ansible]$ ansible-playbook ./playbook4.yml 

PLAY [My Super Puper Playbook for Variables Lesson] ***********************************

TASK [Gathering Facts] ****************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

TASK [Print Secret variable] **********************************************************
ok: [host0] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host1] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host2] => {
    "secret": "HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host1] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host2] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Vladeletc Etogo Servera -->Ivan<--"
}
ok: [host1] => {
    "msg": "Vladeletc Etogo Servera -->Petya<--"
}
ok: [host2] => {
    "msg": "Vladeletc Etogo Servera -->Vasya<--"
}

TASK [set_fact] ***********************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

TASK [debug] **************************************************************************
ok: [host0] => {
    "full_message": "Hello World from Ivan"
}
ok: [host1] => {
    "full_message": "Hello World from Petya"
}
ok: [host2] => {
    "full_message": "Hello World from Vasya"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "ansible_distribution": "CentOS"         <-----Искомая переменная
}
ok: [host1] => {
    "ansible_distribution": "CentOS"         <-----Искомая переменная
}
ok: [host2] => {
    "ansible_distribution": "CentOS"         <-----Искомая переменная
}

PLAY RECAP ****************************************************************************
host0                      : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

Для отображения результата выполнения shell команд добавим в playbook файл следующие строки:
...
  - shell: uptime
      register: results

  - debug:
      var: results
...

В итоге:
...
---
- name: My Super Puper Playbook for Variables Lesson
  hosts: all
  become: yes

  vars:
    message1: Hello
    message2: World
    secret  : HKDN97D6HDN532B

  tasks:

  - name: Print Secret variable
    debug:
      var: secret

  - debug:
      msg: "Sekretnoe slovo: {{ secret }}"

  - debug:
      msg: "Vladeletc Etogo Servera -->{{ owner }}<--"

  - set_fact: full_message="{{ message1 }} {{ message2 }} from {{ owner }}"

  - debug:
      var: full_message

  - debug:
      var: ansible_distribution

  - shell: uptime
    register: results <-----присваивает результат uptime к переменной results

  - debug:
      var: results      <-----выводит на экран переменную results
...

Запускаем:
...
[user@localhost ansible]$ ansible-playbook ./playbook4.yml 

PLAY [My Super Puper Playbook for Variables Lesson] ***********************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host2]
ok: [host0]

TASK [Print Secret variable] **********************************************************
ok: [host0] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host1] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host2] => {
    "secret": "HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host1] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host2] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Vladeletc Etogo Servera -->Ivan<--"
}
ok: [host1] => {
    "msg": "Vladeletc Etogo Servera -->Petya<--"
}
ok: [host2] => {
    "msg": "Vladeletc Etogo Servera -->Vasya<--"
}

TASK [set_fact] ***********************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

TASK [debug] **************************************************************************
ok: [host0] => {
    "full_message": "Hello World from Ivan"
}
ok: [host2] => {
    "full_message": "Hello World from Vasya"
}
ok: [host1] => {
    "full_message": "Hello World from Petya"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "ansible_distribution": "CentOS"
}
ok: [host1] => {
    "ansible_distribution": "CentOS"
}
ok: [host2] => {
    "ansible_distribution": "CentOS"
}

TASK [shell] **************************************************************************
changed: [host1]
changed: [host0]
changed: [host2]

TASK [debug] **************************************************************************
ok: [host0] => {
    "results": {
        "changed": true, 
        "cmd": "uptime", 
        "delta": "0:00:00.011060", 
        "end": "2022-07-31 14:09:00.730753", 
        "failed": false, 
        "rc": 0, 
        "start": "2022-07-31 14:09:00.719693", 
        "stderr": "", 
        "stderr_lines": [], 
        "stdout": " 14:09:00 up 22:07,  1 user,  load average: 0.00, 0.01, 0.05", 
        "stdout_lines": [
            " 14:09:00 up 22:07,  1 user,  load average: 0.00, 0.01, 0.05"
        ]
    }
}
ok: [host1] => {
    "results": {
        "changed": true, 
        "cmd": "uptime", 
        "delta": "0:00:00.014868", 
        "end": "2022-07-31 14:09:00.692973", 
        "failed": false, 
        "rc": 0, 
        "start": "2022-07-31 14:09:00.678105", 
        "stderr": "", 
        "stderr_lines": [], 
        "stdout": " 14:09:00 up 22:07,  1 user,  load average: 0.00, 0.01, 0.05", 
        "stdout_lines": [
            " 14:09:00 up 22:07,  1 user,  load average: 0.00, 0.01, 0.05"
        ]
    }
}
ok: [host2] => {
    "results": {
        "changed": true, 
        "cmd": "uptime", 
        "delta": "0:00:00.012630", 
        "end": "2022-07-31 14:09:00.736519", 
        "failed": false, 
        "rc": 0, 
        "start": "2022-07-31 14:09:00.723889", 
        "stderr": "", 
        "stderr_lines": [], 
        "stdout": " 14:09:00 up 22:06,  1 user,  load average: 0.00, 0.01, 0.05", 
        "stdout_lines": [
            " 14:09:00 up 22:06,  1 user,  load average: 0.00, 0.01, 0.05"
        ]
    }
}

PLAY RECAP ****************************************************************************
host0                      : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

Чтобы не было так много текста, доработаем наш playbook файл, а именно:
...
  - debug:
      var: results.stdout
...

то есть:
...
---
- name: My Super Puper Playbook for Variables Lesson
  hosts: all
  become: yes

  vars:
    message1: Hello
    message2: World
    secret  : HKDN97D6HDN532B

  tasks:

  - name: Print Secret variable
    debug:
      var: secret

  - debug:
      msg: "Sekretnoe slovo: {{ secret }}"

  - debug:
      msg: "Vladeletc Etogo Servera -->{{ owner }}<--"

  - set_fact: full_message="{{ message1 }} {{ message2 }} from {{ owner }}"

  - debug:
      var: full_message

  - debug:
      var: ansible_distribution

  - shell: uptime
    register: results

  - debug:
      var: results.stdout
...

запустим напоследок:
...
[user@localhost ansible]$ ansible-playbook ./playbook4.yml 

PLAY [My Super Puper Playbook for Variables Lesson] ***********************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Print Secret variable] **********************************************************
ok: [host0] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host1] => {
    "secret": "HKDN97D6HDN532B"
}
ok: [host2] => {
    "secret": "HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host2] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}
ok: [host1] => {
    "msg": "Sekretnoe slovo: HKDN97D6HDN532B"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "msg": "Vladeletc Etogo Servera -->Ivan<--"
}
ok: [host1] => {
    "msg": "Vladeletc Etogo Servera -->Petya<--"
}
ok: [host2] => {
    "msg": "Vladeletc Etogo Servera -->Vasya<--"
}

TASK [set_fact] ***********************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

TASK [debug] **************************************************************************
ok: [host0] => {
    "full_message": "Hello World from Ivan"
}
ok: [host1] => {
    "full_message": "Hello World from Petya"
}
ok: [host2] => {
    "full_message": "Hello World from Vasya"
}

TASK [debug] **************************************************************************
ok: [host0] => {
    "ansible_distribution": "CentOS"
}
ok: [host1] => {
    "ansible_distribution": "CentOS"
}
ok: [host2] => {
    "ansible_distribution": "CentOS"
}

TASK [shell] **************************************************************************
changed: [host0]
changed: [host2]
changed: [host1]

TASK [debug] **************************************************************************
ok: [host0] => {
    "results.stdout": " 15:06:24 up 23:05,  1 user,  load average: 0.16, 0.05, 0.06"
}
ok: [host1] => {
    "results.stdout": " 15:06:24 up 23:04,  1 user,  load average: 0.00, 0.01, 0.05"
}
ok: [host2] => {
    "results.stdout": " 15:06:24 up 23:03,  1 user,  load average: 0.08, 0.03, 0.05"
}

PLAY RECAP ****************************************************************************
host0                      : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host1                      : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
host2                      : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

### 12-Ansible - Блоки и Условия – Block-When

Для полноты картины примера сделаем операционную систему host2 Debian. Для этого изменим сам Vagrantfile:
...
[user@localhost ansible]$ vi ./Vagrantfile
...

...
# -*- mode: ruby -*-
# vi: set ft=ruby :

hosts = {
  "host0" => {
    :os => "centos/7",
    :ip => "192.168.1.10"
  },
  "host1" => {
    :os => "centos/7",
    :ip => "192.168.1.11"
  },
  "host2" => {
    :os => "debian/jessie64",
    :ip => "192.168.1.12"
  }
}
ssh_pub_key = File.readlines("/home/user/.ssh/id_rsa.pub").first.strip

Vagrant.configure("2") do |config|
  hosts.each do |name, par|
    config.vm.define name do |machine|
      machine.vm.box = par[:os]
      machine.vm.hostname = "%s" % name
      machine.vm.network "public_network", bridge: "wlp2s0", ip: par[:ip]
      machine.vm.provider "virtualbox" do |v|
        v.name = name
        v.customize ["modifyvm", :id, "--memory", 256]
      end
      machine.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
        echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
        echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
      SHELL
    end
  end
end
...

Запустим эти виртуальные машины:
...
[user@localhost ansible]$ vagrant up
...

Внесём изменение в ansible.cfg, добавив строку "interpreter_python = /usr/bin/python", чтобы не выдавало предупреждение об использовании версии python при запуске ansible, так как у нас один из хостов на Debian:
...
[user@localhost ansible]$ vi ./ansible.cfg
...

...
[defaults]
host_key_checking  = false
inventory          = ./inventory
interpreter_python = /usr/bin/python
...

Inventory файл выглядит так:
...
[staging_servers]
host0 ansible_host=192.168.1.10 ansible_port=22 ansible_user=vagrant ansible_private_key_file=/home/user/.ssh/staging owner=Ivan

[prod_servers]
host1 ansible_host=192.168.1.11 owner=Petya
host2 ansible_host=192.168.1.12 owner=Vasya
...

Создадим новый playbook файл:
...
[user@localhost ansible]$ vi ./playbook5.yml
...

...
---
- name: Install Apache and Upload my Web Page
  hosts: all
  become: yes

  vars:
    source_file: ./MyWebSite/index.html
    destin_file: /var/www/html

  tasks:
  - name: Check and Print LINUX Version
    debug: var=ansible_os_family

  - name: Install Apache Web Server
    yum: name=httpd state=latest

  - name: Copy My Web Page to Servers
    copy: src={{ source_file }} dest={{ destin_file }} mode=0666
    notify: Restart Apache

  - name: Start Apache service and Enable it on the every boot
    service: name=httpd state=started enabled=yes


  handlers:
  - name: Restart Apache
    service: name=httpd state=restarted
...

Запустим ansible-playbook:
...
[user@localhost ansible]$ ansible-playbook ./playbook5.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Check and Print LINUX Version] **************************************************
ok: [host0] => {
    "ansible_os_family": "RedHat"
}
ok: [host1] => {
    "ansible_os_family": "RedHat"
}
ok: [host2] => {
    "ansible_os_family": "Debian"
}

TASK [Install Apache Web Server for RedHat] *******************************************
skipping: [host2]
changed: [host1]
changed: [host0]

TASK [Install Apache Web Server for Debian] *******************************************
skipping: [host0]
skipping: [host1]
changed: [host2]

TASK [Copy My Web Page to Servers] ****************************************************
changed: [host2]
changed: [host0]
changed: [host1]

TASK [Start Apache service and Enable for RedHat] *************************************
skipping: [host2]
changed: [host0]
changed: [host1]

TASK [Start Apache service and Enable for Debian] *************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

PLAY RECAP ****************************************************************************
host0                      : ok=5    changed=3    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
host1                      : ok=5    changed=3    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
host2                      : ok=5    changed=2    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   

[user@localhost ansible]$
...

Как видим, Apache установилось и запустилось на всех серверах. Сможем посмотреть нашу веб страницу, например, на сервере host2 с помощью команды curl:
...
[user@localhost ansible]$ curl 192.168.1.12
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8,10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style.letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout('Elastic()', 108)
}
</SCRIPT>
<BODY bgcolor="gray" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="green"><h2>This WebServer Build by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
</BODY>
</HTML>
[user@localhost ansible]$
...

В playbook файле некоторые команды объединим в блоки:
...
[user@localhost ansible]$ vi ./playbook5.yml
...

...
---
- name: Install Apache and Upload my Web Page
  hosts: all
  become: yes

  vars:
    source_file: ./MyWebSite/index.html
    destin_file: /var/www/html

  tasks:
  - name: Check and Print LINUX Version
    debug: var=ansible_os_family

  - block:    # ===== Block for RedHat ====

    - name: Install Apache Web Server for RedHat
      yum: name=httpd state=latest

    - name: Copy My Web Page to Servers
      copy: src={{ source_file }} dest={{ destin_file }} mode=0666
      notify: Restart Apache RedHat

    - name: Start Apache service and Enable for RedHat
      service: name=httpd state=started enabled=yes

    when: ansible_os_family == "RedHat"

  - block:    # ===== Block for Debian ====

    - name: Install Apache Web Server for Debian
      apt: name=apache2 state=latest

    - name: Copy My Web Page to Servers
      copy: src={{ source_file }} dest={{ destin_file }} mode=0666
      notify: Restart Apache Debian

    - name: Start Apache service and Enable for Debian
      service: name=apache2 state=started enabled=yes

    when: ansible_os_family == "Debian"

  handlers:
  - name: Restart Apache RedHat
    service: name=httpd state=restarted

  - name: Restart Apache Debian
    service: name=apache2 state=restarted
...

Запустим:
...
[user@localhost ansible]$ ansible-playbook ./playbook5.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Check and Print LINUX Version] **************************************************
ok: [host0] => {
    "ansible_os_family": "RedHat"
}
ok: [host1] => {
    "ansible_os_family": "RedHat"
}
ok: [host2] => {
    "ansible_os_family": "Debian"
}

TASK [Install Apache Web Server for RedHat] *******************************************
skipping: [host2]
ok: [host1]
ok: [host0]

TASK [Copy My Web Page to Servers] ****************************************************
skipping: [host2]
ok: [host0]
ok: [host1]

TASK [Start Apache service and Enable for RedHat] *************************************
skipping: [host2]
ok: [host0]
ok: [host1]

TASK [Install Apache Web Server for Debian] *******************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [Copy My Web Page to Servers] ****************************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [Start Apache service and Enable for Debian] *************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

PLAY RECAP ****************************************************************************
host0                      : ok=5    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host1                      : ok=5    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host2                      : ok=5    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   

[user@localhost ansible]$
...
Изменений не произошло, так как всё было установлено и запущено.

Внесём изменение в файл веб страницы index.html, например, изменим цвет фона с серого на голубой:
...
[user@localhost ansible]$ vi ./MyWebSite/index.html
...

...
<BODY bgcolor="blue" onLoad=Elastic()>
...
что получим:
...
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8,10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style.letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout('Elastic()', 108)
}
</SCRIPT>
<BODY bgcolor="blue" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="green"><h2>This WebServer Build by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
</BODY>
</HTML>
...

Снова запустим ansible-playbook:
...
[user@localhost ansible]$ ansible-playbook ./playbook5.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Check and Print LINUX Version] **************************************************
ok: [host0] => {
    "ansible_os_family": "RedHat"
}
ok: [host1] => {
    "ansible_os_family": "RedHat"
}
ok: [host2] => {
    "ansible_os_family": "Debian"
}

TASK [Install Apache Web Server for RedHat] *******************************************
skipping: [host2]
ok: [host1]
ok: [host0]

TASK [Copy My Web Page to Servers] ****************************************************
skipping: [host2]
changed: [host0]
changed: [host1]

TASK [Start Apache service and Enable for RedHat] *************************************
skipping: [host2]
ok: [host0]
ok: [host1]

TASK [Install Apache Web Server for Debian] *******************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [Copy My Web Page to Servers] ****************************************************
skipping: [host0]
skipping: [host1]
changed: [host2]

TASK [Start Apache service and Enable for Debian] *************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

RUNNING HANDLER [Restart Apache RedHat] ***********************************************
changed: [host0]
changed: [host1]

RUNNING HANDLER [Restart Apache Debian] ***********************************************
changed: [host2]

PLAY RECAP ****************************************************************************
host0                      : ok=6    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host1                      : ok=6    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host2                      : ok=6    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   

[user@localhost ansible]$
...
Как видим, изменение веб страницы произошло, проверим с помощью команды curl:
...
[user@localhost ansible]$ curl 192.168.1.12
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8,10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style.letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout('Elastic()', 108)
}
</SCRIPT>
<BODY bgcolor="blue" onLoad=Elastic()> <----- Фон страницы стал голубого цвета
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="green"><h2>This WebServer Build by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
</BODY>
</HTML>
[user@localhost ansible]$
...
Как видим, фон веб страницы стал голубого цвета.

### 13-Ansible - Циклы – Loop, With_Items, Until, With_fileglob

Создадим новый playbook файл:
...
[user@localhost ansible]$ vi ./playbook6.yml
...

...
---
- name: Loops Playbook
  hosts: host1
  become: yes

  tasks:
  - name: Say Hello to All
    debug: msg="Hello, {{ item }}"
    loop:   # для nsible 2.4 и ниже with_item, 2.5 и выше - loop 
      - "Vasya"
      - "Petya" 
      - "Masha" 
      - "Olya"
...

Для простоты примера будем запускать playbook файл на одном из серверов, например, host1.
Запускаем:
...
[user@localhost ansible]$ ansible-playbook ./playbook6.yml 

PLAY [Loops Playbook] *****************************************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]

TASK [Say Hello to All] ***************************************************************
ok: [host1] => (item=Vasya) => {
    "msg": "Hello, Vasya"
}
ok: [host1] => (item=Petya) => {
    "msg": "Hello, Petya"
}
ok: [host1] => (item=Masha) => {
    "msg": "Hello, Masha"
}
ok: [host1] => (item=Olya) => {
    "msg": "Hello, Olya"
}

PLAY RECAP ****************************************************************************
host1                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

В playbook файл добавим строки:
...
  - name: Loop Until example
    shell: echo -n Z >> myfile.txt && cat ./myfile.txt
    register: output  # Сохраняем результат в переменную output
    delay: 2          # Задержка 2 секунды
    retries: 10       # 10 повторений (по умолчанию 3 раза)
    until: output.stdout.find("ZZZZ") == false

  - name: Print Final Output
    debug:
      var: output.stdout
...

В итоге получим:
...
[user@localhost ansible]$ vi ./playbook6.yml
...

...
---
- name: Loops Playbook
  hosts: host1
  become: yes

  tasks:
  - name: Say Hello to All
    debug: msg="Hello, {{ item }}"
    loop:   # для nsible 2.4 и ниже with_item, 2.5 и выше - loop 
      - "Vasya"
      - "Petya"
      - "Masha"
      - "Olya"

  - name: Loop Until example
    shell: echo -n Z >> myfile.txt && cat ./myfile.txt
    register: output  # Сохраняем результат в переменную output
    delay: 2          # Задержка 2 секунды
    retries: 10       # 10 повторений (по умолчанию 3 раза)
    until: output.stdout.find("ZZZZ") == false

  - name: Print Final Output
    debug:
      var: output.stdout
...

Запускаем:
...
[user@localhost ansible]$ ansible-playbook ./playbook6.yml 

PLAY [Loops Playbook] *****************************************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]

TASK [Say Hello to All] ***************************************************************
ok: [host1] => (item=Vasya) => {
    "msg": "Hello, Vasya"
}
ok: [host1] => (item=Petya) => {
    "msg": "Hello, Petya"
}
ok: [host1] => (item=Masha) => {
    "msg": "Hello, Masha"
}
ok: [host1] => (item=Olya) => {
    "msg": "Hello, Olya"
}

TASK [Loop Until example] *************************************************************
FAILED - RETRYING: Loop Until example (10 retries left).
FAILED - RETRYING: Loop Until example (9 retries left).
FAILED - RETRYING: Loop Until example (8 retries left).
changed: [host1]

TASK [Print Final Output] *************************************************************
ok: [host1] => {
    "output.stdout": "ZZZZ"
}

PLAY RECAP ****************************************************************************
host1                      : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

[user@localhost ansible]$
...

Можно зайти на сервер host1 и убедиться, что создался файл myfile.txt:
...
[user@localhost ansible]$ vagrant ssh host1
Last login: Tue Aug  2 15:33:42 2022 from 192.168.1.152
[vagrant@host1 ~]$ ls -l
total 4
-rw-r--r--. 1 root root 4 Aug  2 15:33 myfile.txt
[vagrant@host1 ~]$ cat ./myfile.txt 
ZZZZ[vagrant@host1 ~]$ logout   <---- в начале строки "ZZZZ"
Connection to 127.0.0.1 closed.
[user@localhost ansible]$
...

Можно установить списком приложения:
...
  - name: Install many packages
    yum: name={{ item }} state=installed
    loop:
      - python
      - tree
      - mysql-client
...

Для следующего примера создадим директорию MyWebSite2:
...
[user@localhost ansible]$ mkdir ./MyWebSite2
[user@localhost ansible]$
...

Создадим веб страницу index.html:
...
[user@localhost ansible]$ vi ./MyWebSite2/index.html
...

...
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8, 10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style. letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout ('Elastic()', 100)
}
</SCRIPT>
<BODY bgcolor="gray" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="green"><h2>This COOL WebServer Build by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
<img src="bahamas.png" width = 200 height = 120><img src="bulgaria.png" width = 200 height = 120><br>
<img src="jordan.png" width = 200 height = 120><img src="newzeland.png" width = 200 height = 120><br>
</BODY>
</HTML>
...

В эту же директорию загрузим ещё четыре изображения. В итоге содержимое каталога MyWebSite2:
...
[user@localhost ansible]$ ls -l ./MyWebSite2/
total 28
-rw-r--r--. 1 user user  1387 авг  2 00:27 bahamas.png
-rw-r--r--. 1 user user   556 авг  2 00:27 bulgaria.png
-rw-r--r--. 1 user user   859 авг  2 00:33 index.html
-rw-r--r--. 1 user user  3241 авг  2 00:27 jordan.png
-rw-r--r--. 1 user user 12147 авг  2 00:27 newzeland.png
[user@localhost ansible]$
...

Скопируем playbook5.yml:
...
[user@localhost ansible]$ cp ./playbook5.yml ./playbook7.yml 
[user@localhost ansible]$
...

Внесём изменеия в playbook7.yml:
...
[user@localhost ansible]$ vi ./playbook7.yml
...

...
---
- name: Install Apache and Upload my Web Page
  hosts: all
  become: yes

  vars:
    source_folder: ./MyWebSite2
    destin_folder: /var/www/html

  tasks:
  - name: Check and Print LINUX Version
    debug: var=ansible_os_family

  - block:    # ===== Block for RedHat ====

    - name: Install Apache Web Server for RedHat
      yum: name=httpd state=latest

    - name: Start Apache service and Enable for RedHat
      service: name=httpd state=started enabled=yes

    when: ansible_os_family == "RedHat"

  - block:    # ===== Block for Debian ====

    - name: Install Apache Web Server for Debian
      apt: name=apache2 state=latest

    - name: Start Apache service and Enable for Debian
      service: name=apache2 state=started enabled=yes

    when: ansible_os_family == "Debian"

  - name: Copy My Web Page to Servers
    copy: src={{ source_folder }}/{{ item }} dest={{ destin_folder }} mode=0666
    loop:
      - "index.html"
      - "bahamas.png"
      - "bulgaria.png"
      - "jordan.png"
      - "newzeland.png"
    notify:
      - Restart Apache RedHat
      - Restart Apache Debian

  handlers:
  - name: Restart Apache RedHat
    service: name=httpd state=restarted
    when: ansible_os_family == "RedHat"

  - name: Restart Apache Debian
    service: name=apache2 state=restarted
    when: ansible_os_family == "Debian"
...

Запускаем с playbook7.yml:
...
[user@localhost ansible]$ ansible-playbook ./playbook7.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Check and Print LINUX Version] **************************************************
ok: [host0] => {
    "ansible_os_family": "RedHat"
}
ok: [host1] => {
    "ansible_os_family": "RedHat"
}
ok: [host2] => {
    "ansible_os_family": "Debian"
}

TASK [Install Apache Web Server for RedHat] *******************************************
skipping: [host2]
changed: [host0]
changed: [host1]

TASK [Start Apache service and Enable for RedHat] *************************************
skipping: [host2]
changed: [host0]
changed: [host1]

TASK [Install Apache Web Server for Debian] *******************************************
skipping: [host0]
skipping: [host1]
changed: [host2]

TASK [Start Apache service and Enable for Debian] *************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [Copy My Web Page to Servers] ****************************************************
changed: [host2] => (item=index.html)
changed: [host0] => (item=index.html)
changed: [host1] => (item=index.html)
changed: [host2] => (item=bahamas.png)
changed: [host0] => (item=bahamas.png)
changed: [host1] => (item=bahamas.png)
changed: [host2] => (item=bulgaria.png)
changed: [host1] => (item=bulgaria.png)
changed: [host0] => (item=bulgaria.png)
changed: [host2] => (item=jordan.png)
changed: [host1] => (item=jordan.png)
changed: [host2] => (item=newzeland.png)
changed: [host0] => (item=jordan.png)
changed: [host1] => (item=newzeland.png)
changed: [host0] => (item=newzeland.png)

RUNNING HANDLER [Restart Apache RedHat] ***********************************************
skipping: [host2]
changed: [host1]
changed: [host0]

RUNNING HANDLER [Restart Apache Debian] ***********************************************
skipping: [host1]
skipping: [host0]
changed: [host2]

PLAY RECAP ****************************************************************************
host0                      : ok=6    changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host1                      : ok=6    changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host2                      : ok=6    changed=3    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   

[user@localhost ansible]$
...

Можно зайти на один из серверов, например, host1 и убедиться, что файлы скопировались:
...
[user@localhost ansible]$ vagrant ssh host1
Last login: Tue Aug  2 16:39:41 2022 from 192.168.1.152
[vagrant@host1 ~]$ sudo ls -l /var/www/html/
total 28
-rw-rw-rw-. 1 root root  1387 Aug  2 16:39 bahamas.png
-rw-rw-rw-. 1 root root   556 Aug  2 16:39 bulgaria.png
-rw-rw-rw-. 1 root root   859 Aug  2 16:39 index.html
-rw-rw-rw-. 1 root root  3241 Aug  2 16:39 jordan.png
-rw-rw-rw-. 1 root root 12147 Aug  2 16:39 newzeland.png
[vagrant@host1 ~]$ logout
Connection to 127.0.0.1 closed.
[user@localhost ansible]$
...

Также можно посмотреть с помощью curl нашу веб страницу:
...
[user@localhost ansible]$ curl 192.168.1.11
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8, 10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style. letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout ('Elastic()', 100)
}
</SCRIPT>
<BODY bgcolor="gray" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="green"><h2>This COOL WebServer Build by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
<img src="bahamas.png" width = 200 height = 120><img src="bulgaria.png" width = 200 height = 120><br>
<img src="jordan.png" width = 200 height = 120><img src="newzeland.png" width = 200 height = 120><br>
</BODY>
</HTML>

[user@localhost ansible]$
...
Как видим, наша веб страница отображается.

Секцию копирования 
...
  - name: Copy My Web Page to Servers
    copy: src={{ source_folder }}/{{ item }} dest={{ destin_folder }} mode=0666
    loop:
      - "index.html"
      - "bahamas.png"
      - "bulgaria.png"
      - "jordan.png"
      - "newzeland.png"
    notify:
      - Restart Apache RedHat
      - Restart Apache Debian
...
можно сделать следующим образом
...
  - name: Copy My Web Page to Servers
    copy: src={{ item }} dest={{ destin_folder }} mode=0666
    with_fileglob: "{{ source_folder }}/*.*"
    notify:
      - Restart Apache RedHat
      - Restart Apache Debian
...

В итоге получим playbook7.yml:
...
---
- name: Install Apache and Upload my Web Page
  hosts: all
  become: yes

  vars:
    source_folder: ./MyWebSite2
    destin_folder: /var/www/html

  tasks:
  - name: Check and Print LINUX Version
    debug: var=ansible_os_family

  - block:    # ===== Block for RedHat ====

    - name: Install Apache Web Server for RedHat
      yum: name=httpd state=latest

    - name: Start Apache service and Enable for RedHat
      service: name=httpd state=started enabled=yes

    when: ansible_os_family == "RedHat"

  - block:    # ===== Block for Debian ====

    - name: Install Apache Web Server for Debian
      apt: name=apache2 state=latest

    - name: Start Apache service and Enable for Debian
      service: name=apache2 state=started enabled=yes

    when: ansible_os_family == "Debian"

  - name: Copy My Web Page to Servers
#    copy: src={{ source_folder }}/{{ item }} dest={{ destin_folder }} mode=0666
#    loop:
#      - "index.html"
#      - "bahamas.png"
#      - "bulgaria.png"
#      - "jordan.png"
#      - "newzeland.png"
    copy: src={{ item }} dest={{ destin_folder }} mode=0666
    with_fileglob: "{{ source_folder }}/*.*"
    notify:
      - Restart Apache RedHat
      - Restart Apache Debian

  handlers:
  - name: Restart Apache RedHat
    service: name=httpd state=restarted
    when: ansible_os_family == "RedHat"

  - name: Restart Apache Debian
    service: name=apache2 state=restarted
    when: ansible_os_family == "Debian"
...

Для полноты картины на одном из серверов, например, host1 удалим файлы в директории /var/www/html:
...
[user@localhost ansible]$ vagrant ssh host1
Last login: Tue Aug  2 16:44:47 2022 from 10.0.2.2
[vagrant@host1 ~]$ sudo rm -rf /var/www/html/*
[vagrant@host1 ~]$ ls -l /var/www/html/
total 0
[vagrant@host1 ~]$ logout
Connection to 127.0.0.1 closed.
[user@localhost ansible]$
...

Снова запустим:
...
[user@localhost ansible]$ ansible-playbook ./playbook7.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Check and Print LINUX Version] **************************************************
ok: [host0] => {
    "ansible_os_family": "RedHat"
}
ok: [host1] => {
    "ansible_os_family": "RedHat"
}
ok: [host2] => {
    "ansible_os_family": "Debian"
}

TASK [Install Apache Web Server for RedHat] *******************************************
skipping: [host2]
ok: [host1]
ok: [host0]

TASK [Start Apache service and Enable for RedHat] *************************************
skipping: [host2]
ok: [host1]
ok: [host0]

TASK [Install Apache Web Server for Debian] *******************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [Start Apache service and Enable for Debian] *************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [Copy My Web Page to Servers] ****************************************************
ok: [host2] => (item=/home/user/adv/ansible/./MyWebSite2/index.html)
ok: [host0] => (item=/home/user/adv/ansible/./MyWebSite2/index.html)
changed: [host1] => (item=/home/user/adv/ansible/./MyWebSite2/index.html)
ok: [host2] => (item=/home/user/adv/ansible/./MyWebSite2/bahamas.png)
ok: [host0] => (item=/home/user/adv/ansible/./MyWebSite2/bahamas.png)
changed: [host1] => (item=/home/user/adv/ansible/./MyWebSite2/bahamas.png)
ok: [host2] => (item=/home/user/adv/ansible/./MyWebSite2/bulgaria.png)
ok: [host0] => (item=/home/user/adv/ansible/./MyWebSite2/bulgaria.png)
changed: [host1] => (item=/home/user/adv/ansible/./MyWebSite2/bulgaria.png)
ok: [host2] => (item=/home/user/adv/ansible/./MyWebSite2/jordan.png)
ok: [host0] => (item=/home/user/adv/ansible/./MyWebSite2/jordan.png)
changed: [host1] => (item=/home/user/adv/ansible/./MyWebSite2/jordan.png)
ok: [host2] => (item=/home/user/adv/ansible/./MyWebSite2/newzeland.png)
ok: [host0] => (item=/home/user/adv/ansible/./MyWebSite2/newzeland.png)
changed: [host1] => (item=/home/user/adv/ansible/./MyWebSite2/newzeland.png)

RUNNING HANDLER [Restart Apache RedHat] ***********************************************
changed: [host1]

RUNNING HANDLER [Restart Apache Debian] ***********************************************
skipping: [host1]

PLAY RECAP ****************************************************************************
host0                      : ok=5    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
host1                      : ok=6    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host2                      : ok=5    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   

[user@localhost ansible]$
...

Как видим, на сервер host1 скопировать нужные файлы. Можно убедиться, зайдя на сервер host1:
...
[user@localhost ansible]$ vagrant ssh host1
Last login: Tue Aug  2 17:08:11 2022 from 192.168.1.152
[vagrant@host1 ~]$ sudo ls -l /var/www/html/
total 28
-rw-rw-rw-. 1 root root  1387 Aug  2 17:08 bahamas.png
-rw-rw-rw-. 1 root root   556 Aug  2 17:08 bulgaria.png
-rw-rw-rw-. 1 root root   859 Aug  2 17:08 index.html
-rw-rw-rw-. 1 root root  3241 Aug  2 17:08 jordan.png
-rw-rw-rw-. 1 root root 12147 Aug  2 17:08 newzeland.png
[vagrant@host1 ~]$ logout
Connection to 127.0.0.1 closed.
[user@localhost ansible]$
...

### 14-Ansible - Шаблоны - Jinja Template

Скопируем каталог MyWebSite2:
...
[user@localhost ansible]$ cp -r ./MyWebSite{2,3}
[user@localhost ansible]$
...

В директории изменим имя файла index.html на index.j2:
...
[user@localhost ansible]$ mv ./MyWebSite3/index.{html,j2}
[user@localhost ansible]$
...

Немного подправим содержимое файла index.j2:
...
[user@localhost ansible]$ vi ./MyWebSite3/index.j2
...

...
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8, 10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style. letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout ('Elastic()', 100)
}
</SCRIPT>
<BODY bgcolor="gray" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="gold">Owner of this Server is {{ owner }}<br>
<font color="green"><h2>This COOL WebServer Generated by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
<img src="bahamas.png" width = 200 height = 120><img src="bulgaria.png" width = 200 height = 120><br>
<img src="jordan.png" width = 200 height = 120><img src="newzeland.png" width = 200 height = 120><br>
Server Host Name: {{ ansible_hostname }}<br>
Server OS Family: {{ ansible_os_family }}<br>
IP Address of this Server: {{ ansible_default_ipv4.address }}<br>
</BODY>
</HTML>
...

Создадим наш новый playbook файл:
...
[user@localhost ansible]$ vi ./playbook8.yml
...

...
---
- name: Install Apache and Upload my Web Page
  hosts: all
  become: yes

  vars:
    source_folder: ./MyWebSite3
    destin_folder: /var/www/html

  tasks:
  - name: Check and Print LINUX Version
    debug: var=ansible_os_family

  - block:    # ===== Block for RedHat ====

    - name: Install Apache Web Server for RedHat
      yum: name=httpd state=latest

    - name: Start Apache service and Enable for RedHat
      service: name=httpd state=started enabled=yes

    when: ansible_os_family == "RedHat"

  - block:    # ===== Block for Debian ====

    - name: Install Apache Web Server for Debian
      apt: name=apache2 state=latest

    - name: Start Apache service and Enable for Debian
      service: name=apache2 state=started enabled=yes

    when: ansible_os_family == "Debian"

  - name: Generate INDEX.HTML file
    template: src={{ source_folder }}/index.j2 dest={{ destin_folder }}/index.html mode=0666
    notify:
      - Restart Apache RedHat
      - Restart Apache Debian

  - name: Copy My Web Page to Servers
    copy: src={{ source_folder }}/{{ item }} dest={{ destin_folder }} mode=0666
    loop:
      - "bahamas.png"
      - "bulgaria.png"
      - "jordan.png"
      - "newzeland.png"
    notify:
      - Restart Apache RedHat
      - Restart Apache Debian

  handlers:
  - name: Restart Apache RedHat
    service: name=httpd state=restarted
    when: ansible_os_family == "RedHat"

  - name: Restart Apache Debian
    service: name=apache2 state=restarted
    when: ansible_os_family == "Debian"
...

Запускаем с playbook8.yml:
...
[user@localhost ansible]$ ansible-playbook ./playbook8.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host1]
ok: [host0]
ok: [host2]

TASK [Check and Print LINUX Version] **************************************************
ok: [host0] => {
    "ansible_os_family": "RedHat"
}
ok: [host2] => {
    "ansible_os_family": "Debian"
}
ok: [host1] => {
    "ansible_os_family": "RedHat"
}

TASK [Install Apache Web Server for RedHat] *******************************************
skipping: [host2]
ok: [host0]
ok: [host1]

TASK [Start Apache service and Enable for RedHat] *************************************
skipping: [host2]
ok: [host0]
ok: [host1]

TASK [Install Apache Web Server for Debian] *******************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [Start Apache service and Enable for Debian] *************************************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [Generate INDEX.HTML file] *******************************************************
changed: [host2]
changed: [host1]
changed: [host0]

TASK [Copy My Web Page to Servers] ****************************************************
ok: [host2] => (item=bahamas.png)
ok: [host0] => (item=bahamas.png)
ok: [host1] => (item=bahamas.png)
ok: [host2] => (item=bulgaria.png)
ok: [host0] => (item=bulgaria.png)
ok: [host1] => (item=bulgaria.png)
ok: [host2] => (item=jordan.png)
ok: [host0] => (item=jordan.png)
ok: [host1] => (item=jordan.png)
ok: [host2] => (item=newzeland.png)
ok: [host0] => (item=newzeland.png)
ok: [host1] => (item=newzeland.png)

RUNNING HANDLER [Restart Apache RedHat] ***********************************************
skipping: [host2]
changed: [host1]
changed: [host0]

RUNNING HANDLER [Restart Apache Debian] ***********************************************
skipping: [host0]
skipping: [host1]
changed: [host2]

PLAY RECAP ****************************************************************************
host0                      : ok=7    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host1                      : ok=7    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host2                      : ok=7    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   

[user@localhost ansible]$
...

С помощью curl посмотрим содержимое веб страницы:
...
[user@localhost ansible]$ curl 192.168.1.11
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8, 10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style. letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout ('Elastic()', 100)
}
</SCRIPT>
<BODY bgcolor="gray" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="gold">Owner of this Server is Petya<br> <--- {{ owner }} = Petya
<font color="green"><h2>This COOL WebServer Generated by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
<img src="bahamas.png" width = 200 height = 120><img src="bulgaria.png" width = 200 height = 120><br>
<img src="jordan.png" width = 200 height = 120><img src="newzeland.png" width = 200 height = 120><br>
Server Host Name: host1<br>                 <--- {{ ansible_hostname }} = host1
Server OS Family: RedHat<br>                <--- {{ ansible_os_family }} = RedHat
IP Address of this Server: 192.168.1.11<br> <--- {{ ansible_default_ipv4.address }} = 192.168.1.11
</BODY>
</HTML>
[user@localhost ansible]$
...

Как видим, что переменные в двойных фигурных скобках заменены соответствующими значениями.

### 15-Ansible - Создание Ролей - Roles

Создадим директорий с названием roles (ВАЖНО!) и перейдем в него:
...
[user@localhost ansible]$ mkdir ./roles && cd ./roles
[user@localhost roles]$
...

Запустим волшебную команду ansible-galaxy:
...
[user@localhost roles]$ ansible-galaxy init deploy_apache_web
- Role deploy_apache_web was created successfully
[user@localhost roles]$
...

Эта команда создала директорий deploy_apache_web:
...
[user@localhost roles]$ ls -l
total 0
drwxrwxr-x. 10 user user 154 авг  3 10:22 deploy_apache_web
[user@localhost roles]$
...

со следующей структурой:
...
[user@localhost roles]$ tree
.
└── deploy_apache_web
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml

9 directories, 8 files
[user@localhost roles]$
...

Скопируем png-файлы из директории MyWebSite3 в директорий deploy_apache_web/files:
...
[user@localhost roles]$ cp ../MyWebSite3/*.png ./deploy_apache_web/files/
[user@localhost roles]$
...

Файл index.j2 скопируем в директорий tempates:
...
[user@localhost roles]$ cp ../MyWebSite3/index.j2 ./deploy_apache_web/templates/
[user@localhost roles]$
...

Переносим секции с playbook8.yml по соответстующим файлам main.yml в deploy_apache_web:
...
[user@localhost roles]$ vi ./deploy_apache_web/defaults/main.yml
...
...
---
# defaults file for deploy_apache_web

destin_folder: /var/www/html
...

...
[user@localhost roles]$ vi ./deploy_apache_web/handlers/main.yml
...
...
---
# handlers file for deploy_apache_web

- name: Restart Apache RedHat
  service: name=httpd state=restarted
  when: ansible_os_family == "RedHat"

- name: Restart Apache Debian
  service: name=apache2 state=restarted
  when: ansible_os_family == "Debian"
...

...
[user@localhost roles]$ vi ./deploy_apache_web/tasks/main.yml
...
...
---
# tasks file for deploy_apache_web

- name: Check and Print LINUX Version
  debug: var=ansible_os_family

- block:    # ===== Block for RedHat ====

  - name: Install Apache Web Server for RedHat
    yum: name=httpd state=latest

  - name: Start Apache service and Enable for RedHat
    service: name=httpd state=started enabled=yes

  when: ansible_os_family == "RedHat"

- block:    # ===== Block for Debian ====

  - name: Install Apache Web Server for Debian
    apt: name=apache2 state=latest

  - name: Start Apache service and Enable for Debian
    service: name=apache2 state=started enabled=yes

  when: ansible_os_family == "Debian"

- name: Generate INDEX.HTML file
  template: src=index.j2 dest={{ destin_folder }}/index.html mode=0666
  notify:
    - Restart Apache RedHat
    - Restart Apache Debian

- name: Copy My Web Page to Servers
  copy: src={{ item }} dest={{ destin_folder }} mode=0666
  loop:
    - "bahamas.png"
    - "bulgaria.png"
    - "jordan.png"
    - "newzeland.png"
  notify:
    - Restart Apache RedHat
    - Restart Apache Debian
...

После переноса наш новый playbook файл будет выглядеть так:
...
[user@localhost roles]$ cd .. && vi ./playbook9.yml
...
...
---
- name: Install Apache and Upload my Web Page
  hosts: all
  become: yes

  roles:
    - { role: deploy_apache_web, when: ansible_system == 'Linux' }
...

В этот playbook9.yml добавлено условие, что сработает в том случае, если ОС системы Linux.

В итоге структура директории roles выглядит так:
...
[user@localhost ansible]$ tree ./roles/
./roles/
└── deploy_apache_web
    ├── defaults
    │   └── main.yml
    ├── files
    │   ├── bahamas.png
    │   ├── bulgaria.png
    │   ├── jordan.png
    │   └── newzeland.png
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    │   └── index.j2
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml

9 directories, 13 files
[user@localhost ansible]$
...

Теперь запустим с нашим новым playbook9.yml:
...
[user@localhost ansible]$ ansible-playbook ./playbook9.yml 

PLAY [Install Apache and Upload my Web Page] ******************************************

TASK [Gathering Facts] ****************************************************************
ok: [host0]
ok: [host1]
ok: [host2]

TASK [deploy_apache_web : Check and Print LINUX Version] ******************************
ok: [host0] => {
    "ansible_os_family": "RedHat"
}
ok: [host1] => {
    "ansible_os_family": "RedHat"
}
ok: [host2] => {
    "ansible_os_family": "Debian"
}

TASK [deploy_apache_web : Install Apache Web Server for RedHat] ***********************
skipping: [host2]
changed: [host0]
changed: [host1]

TASK [deploy_apache_web : Start Apache service and Enable for RedHat] *****************
skipping: [host2]
changed: [host1]
changed: [host0]

TASK [deploy_apache_web : Install Apache Web Server for Debian] ***********************
skipping: [host0]
skipping: [host1]
changed: [host2]

TASK [deploy_apache_web : Start Apache service and Enable for Debian] *****************
skipping: [host0]
skipping: [host1]
ok: [host2]

TASK [deploy_apache_web : Generate INDEX.HTML file] ***********************************
changed: [host2]
changed: [host0]
changed: [host1]

TASK [deploy_apache_web : Copy My Web Page to Servers] ********************************
changed: [host2] => (item=bahamas.png)
changed: [host0] => (item=bahamas.png)
changed: [host1] => (item=bahamas.png)
changed: [host2] => (item=bulgaria.png)
changed: [host0] => (item=bulgaria.png)
changed: [host1] => (item=bulgaria.png)
changed: [host2] => (item=jordan.png)
changed: [host0] => (item=jordan.png)
changed: [host1] => (item=jordan.png)
changed: [host2] => (item=newzeland.png)
changed: [host0] => (item=newzeland.png)
changed: [host1] => (item=newzeland.png)

RUNNING HANDLER [deploy_apache_web : Restart Apache RedHat] ***************************
skipping: [host2]
changed: [host0]
changed: [host1]

RUNNING HANDLER [deploy_apache_web : Restart Apache Debian] ***************************
skipping: [host0]
skipping: [host1]
changed: [host2]

PLAY RECAP ****************************************************************************
host0                      : ok=7    changed=5    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host1                      : ok=7    changed=5    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
host2                      : ok=7    changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   

[user@localhost ansible]$
...

Всё готово. С помощью curl можем проверить на всех серверах:
...
[user@localhost ansible]$ curl 192.168.1.10
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8, 10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style. letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout ('Elastic()', 100)
}
</SCRIPT>
<BODY bgcolor="gray" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="gold">Owner of this Server is Ivan<br>
<font color="green"><h2>This COOL WebServer Generated by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
<img src="bahamas.png" width = 200 height = 120><img src="bulgaria.png" width = 200 height = 120><br>
<img src="jordan.png" width = 200 height = 120><img src="newzeland.png" width = 200 height = 120><br>
Server Host Name: host0<br>
Server OS Family: RedHat<br>
IP Address of this Server: 192.168.1.10<br>
</BODY>
</HTML>
[user@localhost ansible]$ 
...

...
[user@localhost ansible]$ curl 192.168.1.11
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8, 10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style. letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout ('Elastic()', 100)
}
</SCRIPT>
<BODY bgcolor="gray" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="gold">Owner of this Server is Petya<br>
<font color="green"><h2>This COOL WebServer Generated by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
<img src="bahamas.png" width = 200 height = 120><img src="bulgaria.png" width = 200 height = 120><br>
<img src="jordan.png" width = 200 height = 120><img src="newzeland.png" width = 200 height = 120><br>
Server Host Name: host1<br>
Server OS Family: RedHat<br>
IP Address of this Server: 192.168.1.11<br>
</BODY>
</HTML>
[user@localhost ansible]$ 
...

...
[user@localhost ansible]$ curl 192.168.1.12
<HTML>
<HEAD>
<TITLE>ADV-IT</TITLE>
<SCRIPT LANGUAGE="JavaScript">
      var sizes = new Array(0,1,2,4,8, 10,12);
      sizes.pos = 0;

function Elastic()
{
    var el = document.all.elastic
    if (null == el.direction)el.direction = 1
    else if ((sizes.pos > sizes.length - 2) || (0 == sizes.pos))
    el.direction *= -1
    el.style. letterSpacing = sizes[sizes.pos += el.direction]
    setTimeout ('Elastic()', 100)
}
</SCRIPT>
<BODY bgcolor="gray" onLoad=Elastic()>
<CENTER>
<br><br><br><br>
<br><br><br><br>
<font color="gold">Owner of this Server is Vasya<br>
<font color="green"><h2>This COOL WebServer Generated by</h2>
<font color="gold"><H1 ID="elastic" ALIGN="Center">ANSIBLE</H1>
<img src="bahamas.png" width = 200 height = 120><img src="bulgaria.png" width = 200 height = 120><br>
<img src="jordan.png" width = 200 height = 120><img src="newzeland.png" width = 200 height = 120><br>
Server Host Name: host2<br>
Server OS Family: Debian<br>
IP Address of this Server: 10.0.2.15<br>
</BODY>
</HTML>
[user@localhost ansible]$ 
...



