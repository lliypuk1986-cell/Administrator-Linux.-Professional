Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.boot_timeout = 1200
  config.ssh.insert_key = false
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Пункт 4: Проброс порта 80 гостя -> 8080 хоста
  config.vm.network "forwarded_port", guest: 80, host: 8080

  # Добавляем два виртуальных диска по 1 ГБ
  config.vm.disk :disk, size: "1GB", name: "disk1"
  config.vm.disk :disk, size: "1GB", name: "disk2"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
    vb.cpus = 1
    vb.gui = false
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
  end

  # ----- Пункт 5: Провижининг -----
  config.vm.provision "shell", inline: <<-SHELL
    # Находим все диски размером 1 ГБ (1073741824 байт) и обрабатываем их
    i=1
    for disk in /dev/sd?; do
      size=$(blockdev --getsize64 $disk)
      if [ "$size" -eq 1073741824 ]; then
        echo "Обработка диска $disk"
        # Форматирование в ext4
        mkfs.ext4 -F $disk
        # Точка монтирования /mnt/disk1, /mnt/disk2
        mount_point="/mnt/disk$i"
        mkdir -p $mount_point
        # Получение UUID
        uuid=$(blkid -s UUID -o value $disk)
        # Добавление в fstab, если ещё не добавлено
        if ! grep -q "UUID=$uuid" /etc/fstab; then
          echo "UUID=$uuid $mount_point ext4 defaults 0 2" >> /etc/fstab
        fi
        # Монтирование
        mount $mount_point
        i=$((i+1))
      fi
    done
    mount -a
    echo "=== Результат монтирования ==="
    df -h | grep /mnt/disk
  SHELL
end