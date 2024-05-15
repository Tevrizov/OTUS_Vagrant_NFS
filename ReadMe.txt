Основная часть: 
vagrant up должен поднимать 2 настроенных виртуальных машины (сервер NFS и клиента) без дополнительных ручных действий;
на сервере NFS должна быть подготовлена и экспортирована директория; 
в экспортированной директории должна быть поддиректория с именем upload с правами на запись в неё; 
экспортированная директория должна автоматически монтироваться на клиенте при старте виртуальной машины (systemd, autofs или fstab — любым способом);
монтирование и работа NFS на клиенте должна быть организована с использованием NFSv3.

1.  Создаю vagrant file
        tep@tep-HYM-WXX:~/OTUS-DZ/11. NFS, FUSE $ cat Vagrantfile 
        Vagrant.configure(2) do |config|
            config.vm.box = "ubuntu/jammy64"
            config.vm.provider "virtualbox" do |v|
            v.memory = 1024
            v.cpus = 1
            end
        
        
            config.vm.define "nfss" do |nfss|
            nfss.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "net1"
            nfss.vm.hostname = "nfss"
            end
        
        
            config.vm.define "nfsc" do |nfsc|
            nfsc.vm.network "private_network", ip: "192.168.50.11", virtualbox__intnet: "net1"
            nfsc.vm.hostname = "nfsc"
            end
        
        
        end
        tep@tep-HYM-WXX:~/OTUS-DZ/11. NFS, FUSE $ 

2. Заускаю ВМ:
        tep@tep-HYM-WXX:~/OTUS-DZ/11. NFS, FUSE $ vagrant status
        Current machine states:

        nfss                      running (virtualbox)
        nfsc                      running (virtualbox)

        This environment represents multiple VMs. The VMs are all listed
        above with their current state. For more information about a specific
        VM, run `vagrant status NAME`.
        tep@tep-HYM-WXX:~/OTUS-DZ/11. NFS, FUSE $ 


3. Настраиваю сервер NFS 
    root@nfss:~# apt install nfs-kernel-server

    Проверяем наличие слушающих портов 2049/udp, 2049/tcp,111/udp, 111/tcp 
    (не все они будут использоваться далее,  но их наличие сигнализирует о том, 
    что необходимые сервисы готовы принимать внешние подключения):
    
    root@nfss:~# ss -tnplu | grep 2049
    tcp   LISTEN 0      64              0.0.0.0:2049       0.0.0.0:*                                                             
    tcp   LISTEN 0      64                 [::]:2049          [::]:*                                                             
    root@nfss:~# ss -tnplu | grep 111
    udp   UNCONN 0      0               0.0.0.0:111        0.0.0.0:*    users:(("rpcbind",pid=2426,fd=5),("systemd",pid=1,fd=93))
    udp   UNCONN 0      0                  [::]:111           [::]:*    users:(("rpcbind",pid=2426,fd=7),("systemd",pid=1,fd=95))
    tcp   LISTEN 0      4096            0.0.0.0:111        0.0.0.0:*    users:(("rpcbind",pid=2426,fd=4),("systemd",pid=1,fd=87))
    tcp   LISTEN 0      4096               [::]:111           [::]:*    users:(("rpcbind",pid=2426,fd=6),("systemd",pid=1,fd=94))
    root@nfss:~# 


4. Создаю и настраиваю директорию, которая будет экспортирована в будущем 
    root@nfss:~# mkdir -p /srv/share/upload 
    root@nfss:~# chown -R nobody:nogroup /srv/share 
    root@nfss:~# chmod 0777 /srv/share/upload 
    root@nfss:~# 


5. Cоздаю в файле /etc/exports структуру, которая позволит экспортировать ранее созданную директорию
    root@nfss:~# cat << EOF > /etc/exports 
    /srv/share 192.168.50.11/32(rw,sync,root_squash)
    EOF
    root@nfss:~# cat /etc/exports 
    /srv/share 192.168.50.11/32(rw,sync,root_squash)
    root@nfss:~# 


6. Экспортирую и проверяю
    root@nfss:~# exportfs -r 
    exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "192.168.50.11/32:/srv/share".
    Assuming default behaviour ('no_subtree_check').
    NOTE: this default has changed since nfs-utils version 1.0.x

    root@nfss:~# exportfs -s 
    /srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
    root@nfss:~# 



7. Настраиваем клиент NFS 
    tep@tep-HYM-WXX:~/OTUS-DZ/11. NFS, FUSE $ vagrant ssh nfsc
    root@nfsc:~# sudo apt install nfs-common


8. Правим /etc/fstab
    root@nfsc:~# echo "192.168.50.10:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" >> /etc/fstab
    root@nfsc:~# ping 192.168.50.10
    PING 192.168.50.10 (192.168.50.10) 56(84) bytes of data.
    64 bytes from 192.168.50.10: icmp_seq=1 ttl=64 time=0.677 ms
    64 bytes from 192.168.50.10: icmp_seq=2 ttl=64 time=0.362 ms
    ^C
    --- 192.168.50.10 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1014ms
    rtt min/avg/max/mdev = 0.362/0.519/0.677/0.157 ms
    root@nfsc:~# 

    и выполняем команды:
    root@nfsc:~# systemctl daemon-reload
    root@nfsc:~# systemctl restart remote-fs.target



9. Проверка работоспособности 
    Создаём тестовый файл на сервере:
    root@nfss:~# cd /srv/share/upload
    root@nfss:/srv/share/upload# ls
    root@nfss:/srv/share/upload# touch check_file
    root@nfss:/srv/share/upload# ls
    check_file
    root@nfss:/srv/share/upload# 

    Проверяем наличие ранее созданного файла:
    root@nfsc:~# cd /mnt/upload/
    root@nfsc:/mnt/upload# ls
    check_file
    root@nfsc:/mnt/upload# 

    Создаём тестовый файл на клиенте:
    root@nfsc:/mnt/upload# touch client_file
    root@nfsc:/mnt/upload# ls
    check_file  client_file  
    root@nfsc:/mnt/upload# 

    Проверяем на сервере:
    root@nfss:/srv/share/upload# ls
    check_file  client_file  client_file.
    root@nfss:/srv/share/upload# pwd
    /srv/share/upload
    root@nfss:/srv/share/upload# 


10. Проверка после перезагрузки
    vagrant reload
    vagrant status
    vagrant ssh nfsc
    vagrant ssh nfss


    -client
    vagrant@nfsc:~$ sudo -i
    root@nfsc:~# cd /mnt/upload/
    root@nfsc:/mnt/upload# ls
    check_file  client_file  
    root@nfsc:/mnt/upload# 

    root@nfsc:/mnt/upload# showmount -a 192.168.50.10
    All mount points on 192.168.50.10:
    192.168.50.11:/srv/share
    root@nfsc:/mnt/upload# 
    
    root@nfsc:/mnt/upload#  mount | grep mnt
    systemd-1 on /mnt type autofs (rw,relatime,fd=60,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=15691)
    192.168.50.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=43044,mountproto=udp,local_lock=none,addr=192.168.50.10)
    nsfs on /run/snapd/ns/lxd.mnt type nsfs (rw)
    root@nfsc:/mnt/upload# touch final_check;
    

    -server
    vagrant@nfss:~$ sudo -i
    root@nfss:~# cd /srv/share/upload
    root@nfss:/srv/share/upload# ls
    check_file  client_file  client_file.
    root@nfss:/srv/share/upload# 
    root@nfss:/srv/share/upload# exportfs -s
    /srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
    root@nfss:/srv/share/upload# 

    root@nfss:/srv/share/upload# showmount -a 192.168.50.10
    All mount points on 192.168.50.10:
    192.168.50.11:/srv/share
    root@nfss:/srv/share/upload# 

    root@nfss:/srv/share/upload# ls
    check_file  client_file  final_check
    root@nfss:/srv/share/upload# 


-------------------------------------------------------------------------------------------------------


Создание автоматизированного Vagrantfile 

Vagrantfile - /newvagrantfile/Vagrantfile
playbook - /newvagrantfile/playbook.yml/ Подробное описание изложил в этом файле.



