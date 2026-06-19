# Версия хостовой ОС Windows 11 Pro, используется WSL (Windows Subsystem for Linux) 
# Версия Vagrant 2.4.3
# Версия VirtualBox 7.1.6
# Скачиваем box для Ubuntu 22.04 https://app.vagrantup.com/ubuntu/boxes/jammy64/versions/20241002.0.0/providers/virtualbox.box
# Создаем на хостовой машине папку C:\vagrant_projects\my_vm\vagrant,  копируем в нее box 
# Создаем Vagrantfile 
Vagrant.configure("2") do |config|
  # Ubuntu Jammy
  config.vm.define "ubuntu-jammy" do |srv|
    srv.vm.box = "ubuntu/jammy64"
    srv.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 2
      vb.name = "ubuntu-jammy-vm"
      vb.customize ['modifyvm', :id, '--audio', 'none']
    end

    # Синхронизация папок
    srv.vm.synced_folder "./", "/vagrant"

    # Подключение диска
    srv.vm.disk :disk, size: "1GB", name: "disk1"
    srv.vm.disk :disk, size: "1GB", name: "disk2"
    # Проброс порта для HTTP (будет доступен по localhost:8080)
    srv.vm.network(:forwarded_port,
                    guest: 80,
                    host: 8080,
                    host_ip: "127.0.0.1")

    # Приватная сеть в той же подсети, что и virbr0 на хосте (192.168.56.0/21)
    srv.vm.network(:private_network,
                   ip: "192.168.56.14",
                   libvirt__network_name: "192.168.56.14",
                   auto_config: true)

    # Provisioning - форматирует добавленные диски в файловую систему ext4
    srv.vm.provision "shell", inline: <<-SHELL
       # Форматируем диски через shell
    srv.vm.provision "shell", inline: <<-SHELL
    # Проверяем и форматируем первый диск
    if [ -b /dev/sdb ]; then
      echo "Formatting /dev/sdb as ext4..."
      sudo mkfs.ext4 -F /dev/sdb
    else
      echo "Disk /dev/sdb not found"
    fi

    # Проверяем и форматируем второй диск
    if [ -b /dev/sdc ]; then
      echo "Formatting /dev/sdc as ext4..."
      sudo mkfs.ext4 -F /dev/sdc
    else
      echo "Disk /dev/sdc not found"
    fi

    # Создаем точки монтирования
    sudo mkdir -p /mnt/disk1
    sudo mkdir -p /mnt/disk2

    # Монтируем диски
    sudo mount /dev/sdb /mnt/disk1
    sudo mount /dev/sdc /mnt/disk2

    # Добавляем в fstab для автоматического монтирования
    echo "/dev/sdb /mnt/disk1 ext4 defaults 0 0" | sudo tee -a /etc/fstab
    echo "/dev/sdc /mnt/disk2 ext4 defaults 0 0" | sudo tee -a /etc/fstab
    SHELL
  end
end

# В терминале WSL переходим в папку cd /mnt/c/vagrant_projects/my_vm/vagrant
ubuntuadmin@WS01:~$ cd /mnt/c/vagrant_projects/my_vm/vagrant
ubuntuadmin@WS01:/mnt/c/vagrant_projects/my_vm/vagrant$
# Добавляем box
vagrant box add  ubuntu-jammy64.box --name ubuntu-jammy64
ubuntuadmin@WS01:/mnt/c/vagrant_projects/my_vm/vagrant$ vagrant box add  ubuntu-jammy64.box --name ubuntu-jammy64
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'ubuntu-jammy64' (v0) for provider:
    box: Unpacking necessary files from: file:///mnt/c/vagrant_projects/my_vm/vagrant/ubuntu-jammy64.box
==> box: Successfully added box 'ubuntu-jammy64' (v0) for ''!
# Создаем каталог  ~/vagrant_project/vagrant
ubuntuadmin@WS01:/mnt/c/vagrant_projects/my_vm/vagrant$ mkdir ~/vagrant_project/vagrant
# Скопировать проект в WSL
ubuntuadmin@WS01:/mnt/c/vagrant_projects/my_vm/vagrant$ cp -r /mnt/c/vagrant_projects/vagrant ~/vagrant_project/vagrant
# Перейдем в проект
ubuntuadmin@WS01:/mnt/c/vagrant_projects/my_vm/vagrant$ cd ~/vagrant_project/vagrant/
ubuntuadmin@WS01:~/vagrant_project/vagrant$ ls
vagrant
ubuntuadmin@WS01:~/vagrant_project/vagrant$ cd vagrant/
ubuntuadmin@WS01:~/vagrant_project/vagrant/vagrant$ l
