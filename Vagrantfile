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

  # Провижининг (С защитой от мелких кэшированных дисков)
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
