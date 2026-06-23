# Версия хостовой ОС Windows 11 Pro, используется WSL (Windows Subsystem for Linux) 
# Версия Vagrant 2.4.3
# Версия VirtualBox 7.1.6
# Скачиваем box для Ubuntu 22.04 https://app.vagrantup.com/ubuntu/boxes/jammy64/versions/20241002.0.0/providers/virtualbox.box
# Создаем на хостовой машине папку C:\vagrant_projects\my_vm\vagrant\vagrant,  копируем в нее box 
# Создаем Vagrantfile 
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.box_url = "file://C:/vagrant_projects/my_vm/vagrant/vagrant/ubuntu-jammy64.box"
  config.vm.boot_timeout = 600

  config.vm.define "ubuntu10" do |ubuntu10|
    ubuntu10.vm.hostname = "ubuntu10"
    
    ubuntu10.vm.network "private_network", ip: "192.168.56.10"
    ubuntu10.vm.network "forwarded_port", guest: 80, host: 8080, id: "web_port"
    
    ubuntu10.ssh.insert_key = false
    ubuntu10.vm.synced_folder ".", "/vagrant", disabled: true

    # ИЗМЕНЕНИЕ: Меняем имена дисков на data_disk_, чтобы сбросить глобальный кэш VirtualBox
    ubuntu10.vm.disk :disk, name: "data_disk_1", size: "1GB"
    ubuntu10.vm.disk :disk, name: "data_disk_2", size: "1GB"
  end

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = 1024
    vb.cpus = 2
    vb.name = "ubuntu10"
  end

   #  Провижининг (С защитой от мелких кэшированных дисков)
  config.vm.provision "shell", inline: <<-SHELL
    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y parted net-tools

    echo "=== НАСТРОЙКА ДИСКОВ ==="
    # ИСПРАВЛЕНИЕ: Скрипт выбирает только те диски, чей размер БОЛЬШЕ 500 Мегабайт (исключая sda и loop)
    # Это заставит Ubuntu полностью проигнорировать фантомный диск 10M
    VALID_DISKS=$(lsblk -dn -o NAME,SIZE -b | grep -v "sda" | grep -v "loop" | awk '$2 > 524288000 {print $1}')

    for disk in $VALID_DISKS; do
      full_disk="/dev/$disk"
      if [ -b "$full_disk" ]; then
        if ! blkid "${full_disk}1" >/dev/null 2>&1; then
          echo "Найден валидный диск: $full_disk. Размечаем..."
          parted -s "$full_disk" mklabel gpt
          parted -s "$full_disk" mkpart primary ext4 1MiB 100%
          udevadm settle
          mkfs.ext4 -F "${full_disk}1"
        else
          echo "Диск $full_disk уже размечен."
        fi
      fi
    done

    # Очищаем старые неверные записи в fstab, если они там были
    sed -i '/mnt\/disk/d' /etc/fstab

    # Динамически находим новые разделы размером ~1ГБ и монтируем их
    PARTITIONS=$(lsblk -ln -o NAME,TYPE,SIZE -b | grep "part" | grep -v "sda1" | awk '$3 > 524288000 {print $1}')
    
    COUNTER=1
    for part in $PARTITIONS; do
      mkdir -p "/mnt/disk$COUNTER"
      if mount "/dev/$part" "/mnt/disk$COUNTER" 2>/dev/null; then
        echo "/dev/$part /mnt/disk$COUNTER ext4 defaults,nofail 0 2" >> /etc/fstab
        echo "Успешно: /dev/$part смонтирован в /mnt/disk$COUNTER"
      fi
      COUNTER=$((COUNTER + 1))
    done

    echo "=== ИТОГОВАЯ ПРОВЕРКА ==="
    echo "--- Блочные устройства ---"
    lsblk
    echo "--- Смонтированные папки ---"
    df -h | grep disk
  SHELL
end


# В терминале Poweshell  переходим в папку cd C:\vagrant_projects\my_vm\vagrant\vagrant
# Запускаем vagrant up
PS C:\vagrant_projects\my_vm\vagrant\vagrant> vagrant up
Bringing machine 'ubuntu10' up with 'virtualbox' provider...
==> ubuntu10: Importing base box 'ubuntu/jammy64'...
==> ubuntu10: Matching MAC address for NAT networking...
==> ubuntu10: Checking if box 'ubuntu/jammy64' version '20241002.0.0' is up to date...
==> ubuntu10: Setting the name of the VM: ubuntu10
==> ubuntu10: Clearing any previously set network interfaces...
==> ubuntu10: Preparing network interfaces based on configuration...
    ubuntu10: Adapter 1: nat
    ubuntu10: Adapter 2: hostonly
==> ubuntu10: Forwarding ports...
    ubuntu10: 80 (guest) => 8080 (host) (adapter 1)
    ubuntu10: 22 (guest) => 2222 (host) (adapter 1)
==> ubuntu10: Configuring storage mediums...
    ubuntu10: Disk 'data_disk_1' not found in guest. Creating and attaching disk to guest...
    ubuntu10: Disk 'data_disk_2' not found in guest. Creating and attaching disk to guest...
==> ubuntu10: Running 'pre-boot' VM customizations...
==> ubuntu10: Booting VM...
==> ubuntu10: Waiting for machine to boot. This may take a few minutes...
    ubuntu10: SSH address: 127.0.0.1:2222
    ubuntu10: SSH username: vagrant
    ubuntu10: SSH auth method: private key
    ubuntu10: Warning: Connection aborted. Retrying...
    ubuntu10: Warning: Connection reset. Retrying...
==> ubuntu10: Machine booted and ready!
==> ubuntu10: Checking for guest additions in VM...
    ubuntu10: The guest additions on this VM do not match the installed version of
    ubuntu10: VirtualBox! In most cases this is fine, but in rare cases it can
    ubuntu10: prevent things such as shared folders from working properly. If you see
    ubuntu10: shared folder errors, please make sure the guest additions within the
    ubuntu10: virtual machine match the version of VirtualBox you have installed on
    ubuntu10: your host and reload your VM.
    ubuntu10:
    ubuntu10: Guest Additions Version: 6.0.0 r127566
    ubuntu10: VirtualBox Version: 7.1
==> ubuntu10: Setting hostname...
==> ubuntu10: Configuring and enabling network interfaces...
==> ubuntu10: Running provisioner: shell...
    ubuntu10: Running: inline script
    ubuntu10: Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
    ubuntu10: Get:2 http://security.ubuntu.com/ubuntu jammy-security InRelease [129 kB]
    ubuntu10: Get:3 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]
    ubuntu10: Get:4 http://archive.ubuntu.com/ubuntu jammy-backports InRelease [127 kB]
    ubuntu10: Get:5 http://security.ubuntu.com/ubuntu jammy-security/main amd64 Packages [3311 kB]
    ubuntu10: Get:6 http://archive.ubuntu.com/ubuntu jammy/universe amd64 Packages [14.1 MB]
    ubuntu10: Get:7 http://security.ubuntu.com/ubuntu jammy-security/main Translation-en [466 kB]
    ubuntu10: Get:8 http://security.ubuntu.com/ubuntu jammy-security/main amd64 c-n-f Metadata [14.5 kB]
    ubuntu10: Get:9 http://security.ubuntu.com/ubuntu jammy-security/restricted amd64 Packages [5901 kB]
    ubuntu10: Get:10 http://security.ubuntu.com/ubuntu jammy-security/restricted Translation-en [1125 kB]
    ubuntu10: Get:11 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 Packages [1040 kB]
    ubuntu10: Get:12 http://security.ubuntu.com/ubuntu jammy-security/universe Translation-en [232 kB]
    ubuntu10: Get:13 http://security.ubuntu.com/ubuntu jammy-security/universe amd64 c-n-f Metadata [23.1 kB]
    ubuntu10: Get:14 http://security.ubuntu.com/ubuntu jammy-security/multiverse amd64 Packages [64.3 kB]
    ubuntu10: Get:15 http://security.ubuntu.com/ubuntu jammy-security/multiverse Translation-en [12.6 kB]
    ubuntu10: Get:16 http://security.ubuntu.com/ubuntu jammy-security/multiverse amd64 c-n-f Metadata [544 B]
    ubuntu10: Get:17 http://archive.ubuntu.com/ubuntu jammy/universe Translation-en [5652 kB]
    ubuntu10: Get:18 http://archive.ubuntu.com/ubuntu jammy/universe amd64 c-n-f Metadata [286 kB]
    ubuntu10: Get:19 http://archive.ubuntu.com/ubuntu jammy/multiverse amd64 Packages [217 kB]
    ubuntu10: Get:20 http://archive.ubuntu.com/ubuntu jammy/multiverse Translation-en [112 kB]
    ubuntu10: Get:21 http://archive.ubuntu.com/ubuntu jammy/multiverse amd64 c-n-f Metadata [8372 B]
    ubuntu10: Get:22 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [3579 kB]
    ubuntu10: Get:23 http://archive.ubuntu.com/ubuntu jammy-updates/main Translation-en [536 kB]
    ubuntu10: Get:24 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 c-n-f Metadata [19.9 kB]
    ubuntu10: Get:25 http://archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 Packages [6120 kB]
    ubuntu10: Get:26 http://archive.ubuntu.com/ubuntu jammy-updates/restricted Translation-en [1166 kB]
    ubuntu10: Get:27 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [1276 kB]
    ubuntu10: Get:28 http://archive.ubuntu.com/ubuntu jammy-updates/universe Translation-en [320 kB]
    ubuntu10: Get:29 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 c-n-f Metadata [30.7 kB]
    ubuntu10: Get:30 http://archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 Packages [71.6 kB]
    ubuntu10: Get:31 http://archive.ubuntu.com/ubuntu jammy-updates/multiverse Translation-en [15.5 kB]
    ubuntu10: Get:32 http://archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 c-n-f Metadata [756 B]
    ubuntu10: Get:33 http://archive.ubuntu.com/ubuntu jammy-backports/main amd64 Packages [70.2 kB]
    ubuntu10: Get:34 http://archive.ubuntu.com/ubuntu jammy-backports/main Translation-en [11.4 kB]
    ubuntu10: Get:35 http://archive.ubuntu.com/ubuntu jammy-backports/main amd64 c-n-f Metadata [412 B]
    ubuntu10: Get:36 http://archive.ubuntu.com/ubuntu jammy-backports/restricted amd64 c-n-f Metadata [116 B]
    ubuntu10: Get:37 http://archive.ubuntu.com/ubuntu jammy-backports/universe amd64 Packages [30.8 kB]
    ubuntu10: Get:38 http://archive.ubuntu.com/ubuntu jammy-backports/universe Translation-en [16.9 kB]
    ubuntu10: Get:39 http://archive.ubuntu.com/ubuntu jammy-backports/universe amd64 c-n-f Metadata [676 B]
    ubuntu10: Get:40 http://archive.ubuntu.com/ubuntu jammy-backports/multiverse amd64 c-n-f Metadata [116 B]
    ubuntu10: Fetched 46.2 MB in 27s (1715 kB/s)
    ubuntu10: Reading package lists...
    ubuntu10: Reading package lists...
    ubuntu10: Building dependency tree...
    ubuntu10: Reading state information...
    ubuntu10: parted is already the newest version (3.4-2build1).
    ubuntu10: parted set to manually installed.
    ubuntu10: The following NEW packages will be installed:
    ubuntu10:   net-tools
    ubuntu10: 0 upgraded, 1 newly installed, 0 to remove and 6 not upgraded.
    ubuntu10: Need to get 204 kB of archives.
    ubuntu10: After this operation, 819 kB of additional disk space will be used.
    ubuntu10: Get:1 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 net-tools amd64 1.60+git20181103.0eebece-1ubuntu5.4 [204 kB]
    ubuntu10: Fetched 204 kB in 1s (179 kB/s)
    ubuntu10: Selecting previously unselected package net-tools.
(Reading database ... 64286 files and directories currently installed.)
    ubuntu10: Preparing to unpack .../net-tools_1.60+git20181103.0eebece-1ubuntu5.4_amd64.deb ...
    ubuntu10: Unpacking net-tools (1.60+git20181103.0eebece-1ubuntu5.4) ...
    ubuntu10: Setting up net-tools (1.60+git20181103.0eebece-1ubuntu5.4) ...
    ubuntu10: Processing triggers for man-db (2.10.2-1) ...
    ubuntu10:
    ubuntu10: Running kernel seems to be up-to-date.
    ubuntu10:
    ubuntu10: No services need to be restarted.
    ubuntu10:
    ubuntu10: No containers need to be restarted.
    ubuntu10:
    ubuntu10: No user sessions are running outdated binaries.
    ubuntu10:
    ubuntu10: No VM guests are running outdated hypervisor (qemu) binaries on this host.
    ubuntu10: === НАСТРОЙКА ДИСКОВ ===
    ubuntu10: Найден валидный диск: /dev/sdc. Размечаем...
    ubuntu10: mke2fs 1.46.5 (30-Dec-2021)
    ubuntu10: Creating filesystem with 261632 4k blocks and 65408 inodes
    ubuntu10: Filesystem UUID: 87dea5b5-1ad0-4739-bd4d-ee0e8ab7067f
    ubuntu10: Superblock backups stored on blocks:
    ubuntu10:   32768, 98304, 163840, 229376
    ubuntu10:
    ubuntu10: Allocating group tables: done
    ubuntu10: Writing inode tables: done
    ubuntu10: Creating journal (4096 blocks): done
    ubuntu10: Writing superblocks and filesystem accounting information: done
    ubuntu10:
    ubuntu10: Найден валидный диск: /dev/sdd. Размечаем...
    ubuntu10: mke2fs 1.46.5 (30-Dec-2021)
    ubuntu10: Creating filesystem with 261632 4k blocks and 65408 inodes
    ubuntu10: Filesystem UUID: 9421e393-b054-4283-8174-16eec5e51765
    ubuntu10: Superblock backups stored on blocks:
    ubuntu10:   32768, 98304, 163840, 229376
    ubuntu10:
    ubuntu10: Allocating group tables: done
    ubuntu10: Writing inode tables: done
    ubuntu10: Creating journal (4096 blocks): done
    ubuntu10: Writing superblocks and filesystem accounting information: done
    ubuntu10:
    ubuntu10: sed: -e expression #1, char 7: extra characters after command
    ubuntu10: Успешно: /dev/sdc1 смонтирован в /mnt/disk1
    ubuntu10: Успешно: /dev/sdd1 смонтирован в /mnt/disk2
    ubuntu10: === ИТОГОВАЯ ПРОВЕРКА ===
    ubuntu10: --- Блочные устройства ---
    ubuntu10: NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
    ubuntu10: loop0    7:0    0 63.8M  1 loop /snap/core20/2866
    ubuntu10: loop1    7:1    0 91.7M  1 loop /snap/lxd/38800
    ubuntu10: loop2    7:2    0 49.3M  1 loop /snap/snapd/26865
    ubuntu10: sda      8:0    0   40G  0 disk
    ubuntu10: └─sda1   8:1    0   40G  0 part /
    ubuntu10: sdb      8:16   0   10M  0 disk
    ubuntu10: sdc      8:32   0    1G  0 disk
    ubuntu10: └─sdc1   8:33   0 1022M  0 part /mnt/disk1
    ubuntu10: sdd      8:48   0    1G  0 disk
    ubuntu10: └─sdd1   8:49   0 1022M  0 part /mnt/disk2
    ubuntu10: --- Смонтированные папки ---
    ubuntu10: /dev/sdc1       988M   24K  921M   1% /mnt/disk1
    ubuntu10: /dev/sdd1       988M   24K  921M   1% /mnt/disk2
    # Подключаемся vagrant ssh
    PS C:\vagrant_projects\my_vm\vagrant\vagrant> vagrant ssh
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-181-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Tue Jun 23 17:49:13 UTC 2026

  System load:             0.7
  Usage of /:              4.6% of 38.70GB
  Memory usage:            21%
  Swap usage:              0%
  Processes:               115
  Users logged in:         0
  IPv4 address for enp0s3: 10.0.2.15
  IPv6 address for enp0s3: fd00::24:d8ff:fe7c:d655


Expanded Security Maintenance for Applications is not enabled.

6 updates can be applied immediately.
6 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '24.04.4 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


vagrant@ubuntu10:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            96M 1008K   95M   2% /run
/dev/sda1        39G  1.8G   37G   5% /
tmpfs           479M     0  479M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/sdc1       988M   24K  921M   1% /mnt/disk1
/dev/sdd1       988M   24K  921M   1% /mnt/disk2
tmpfs            96M  4.0K   96M   1% /run/user/1000
vagrant@ubuntu10:~$
# Скриншот вывода команды df -h из ВМ
<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/c6162e9e-ff83-4764-843e-ffd942547874" />
# Скриншот проверки проброса порта на хостовой ВМ
<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/09fb7923-376d-438e-85fd-84861dc3131f" />
PS C:\vagrant_projects\my_vm\vagrant\vagrant> Get-NetTCPConnection -LocalPort 8080 | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess


LocalAddress  : 0.0.0.0
LocalPort     : 8080
RemoteAddress : 0.0.0.0
RemotePort    : 0
State         : Listen
OwningProcess : 12980



PS C:\vagrant_projects\my_vm\vagrant\vagrant> netstat -ano | findstr 8080
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       12980


