# Перенос сервера Linux с физического тома на LVM том без потери данных
 
1. Создаём минимум 2 раздела: корень и бут на новом диске:

        fdisk /dev/sdb

1.1. создаём первый том (загрузочный) на 500МБ

![image](https://user-images.githubusercontent.com/40124505/227447973-6e5dd9ef-4c4f-4b66-95c8-4cdc4b04395e.png)

1.2. назначаем его загрузочным:

![image](https://user-images.githubusercontent.com/40124505/227448522-31f73890-6bb0-4e2e-8bf0-0c3d548a0c5b.png)

1.3. создаём корневой раздел:

![image](https://user-images.githubusercontent.com/40124505/227448756-bdc53848-a6ea-43dc-9689-9847ce69f4c0.png)

1.4. меняем корневой том на LVM:

![image](https://user-images.githubusercontent.com/40124505/227449218-52f28a41-9421-42bb-ac53-860c3a345e44.png)

2. Инициализируем LVM том:

        Pvcreate /dev/sdb2

``проверить можно командой pvdisplay``

2.1. создаём виртуальную группу томов:

      Vgcreate centest /dev/sdX2

``где "centest" это любое желаеммое имя группы`` ``проверить можно командой vgdisplay``

2.2. (Опционально) создаём том подкачки на LVM на 10ГБ:

    lvcreate -L10G -n swap centest
    
2.3. Создаём корневой том на всё оствашееся пространство:
    
    lvcreate –l 100%FREE –n root contest

``ОЧЕНЬ важно корневой раздел назвать root, иначе система не сможет загрузиться``

2.4. создаём файловую систему на всех томах:

    mkfs.ext4 /dev/sdb1 && mkfs.ext4 /dev/centest/swap && mkfs.ext4 /dev/centest/root
    
3. Монтируем наш корень во временную папку mnt и создаём там директорию boot:

        mount /dev/centest/root /mnt && mkdir /mnt/boot

3.1. монтируем наш загрузочный том:

      mount /dev/sdb1 /mnt/boot
      
3.2. копируем всё содержимое нашего сервера на новый диск:

      rsync -aAXv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"}  /  /mnt/
      
3.3. теперь мы связываем директории нынешнего корня и монтированного:
 
      mount --bind /dev  /mnt/dev && mount --bind /proc  /mnt/proc && mount --bind /sys  /mnt/sys

3.4. Заходим в наш корень через chroot:
     
      chroot /mnt

3.5. сейчас нужно узнать  UUID загрузочного тома:

      blkid /dev/sdb1
      
3.6. открываем fstab:

      nano /etc/fstab
      
3.7. вводим аналогично изображению:

![image](https://user-images.githubusercontent.com/40124505/227457990-33cec49b-bb60-4d4b-909a-2b03abb1ce10.png)

``UUID указываем свой`` ``Если раздел swap не создавали, то лучше его убрать из файла, иначе возможно будет долгая загрузка OC``

4. Сейчас нам нужно узнать, поддерживает ли наш initramfs тома LVM:

        lsinitrd  | grep lvm |wc -l 
      
4.1. Если выдало 0, значит пересобираем initranfs:

        dracut -a lvm --fstab --force 
        
4.2. теперь нужно обновить конфиг загрузчика:

        grub2-mkconfig –o /boot/grub2/grub.cfg
        
4.3. (Опционально)  Есть вероятность, что загрузчик не увидит систему, потому что при обновлении конфига система располагалась на Sdb, а не sda.

В этомм случае нужно открыть конфиг:

    nano /boot/grub2/grub.cfg
   
И найти строку:

    set root = “hd1,msdos1"
    
Её нужно изменить на:

    set root = “hd0,msdos1"
