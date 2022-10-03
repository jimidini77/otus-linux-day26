# otus-linux-day26
## *Backups*

# **Prerequisite**
- Host OS: Debian 11.3.0
- Guest OS: CentOS 7.8.2003
- VirtualBox: 6.1.36
- Vagrant: 2.2.19
- Ansible: 2.12.4

# **Содержание ДЗ**

Настроить стенд Vagrant с двумя виртуальными машинами: `backup` и `client`.
Настроить удаленный бэкап каталога /etc c сервера client при помощи `borgbackup`. Резервные копии должны соответствовать следующим критериям:
- директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;
- репозиторий для резервных копий должен быть зашифрован ключом или паролем;
- имя бэкапа должно содержать информацию о времени снятия бекапа;
- глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день;
- резервная копия снимается каждые 5 минут;
- написан скрипт для снятия резервных копий. Скрипт запускается из systemd timer;
- настроено логирование процесса бекапа.

# **Выполнение**

Созданы 2 ВМ `client`, `backup` к машине для хранения бэкапов подключен дополнительный диск:
```ruby
Vagrant.configure("2") do |config|
config.vm.synced_folder ".", "/vagrant", disabled: true
file_to_disk = './sata1.vdi'

  config.vm.define "backup" do |backup|
    backup.vm.box = "centos/7"
    backup.vm.network "private_network", ip: "192.168.56.160"
    backup.vm.hostname = "backup"
    backup.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
      needsController = false
      unless File.exist?(file_to_disk)
        vb.customize ['createhd', '--filename', file_to_disk, '--variant', 'Fixed', '--size', 2048]
        needsController =  true
      end
      if needsController == true
        vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
        vb.customize ['storageattach', :id, '--storagectl', 'SATA', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', file_to_disk]
     end
    end
  end

  config.vm.define "client" do |client|
    client.vm.box = "centos/7"
    client.vm.network "private_network", ip: "192.168.56.150"
    client.vm.hostname = "client"
    client.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "512"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
    end
    client.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/provision.yml"
      ansible.inventory_path = "ansible/hosts"
      ansible.host_key_checking = "false"
      ansible.limit = "all"
      ansible.verbose = "true"
    end
  end
end
```
Далее приведены выдержки содержимого playbook Ansible.

На обе машины подключен репозиторий `epel-release` и установлен `borgbackup`:
```yml
- name: SW CONFIG | Install and configure software
  hosts: all
  become: true

  tasks:
    - name: SW CONFIG | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present

    - name: SW CONFIG | Install BORGBACKUP package from EPEL Repo
      yum:
        name: borgbackup
        state: latest
```
На машине клиенте генерируется ключевая пара клиента для SSH:
```yml
    - name: SSH KEYPAIR GENERATION
      openssh_keypair:
        path: /root/.ssh/id_rsa
        size: 2048
```
Помещается публичный ключ `backup` сервера в список известных:
```yml
    - name: Tell client about backup server
      shell: ssh-keyscan -H 192.168.56.160 >> ~/.ssh/known_hosts
```
На сервере `backup` готовится диск для хранения бэкапов, выполняется создание раздела, форматирование, монтирование, имя диска выбирается динамически:
```yml
- name: BACKUP CONFIG | Disk Configuration
  hosts: backup
  become: true
  tasks:
    - name: BACKUP CONFIG | Select disk name
      shell: if [[ $(lsblk | grep sda1) ]]; then echo 'sdb'; else echo 'sda'; fi
      register: backup_disk

    - name: BACKUP CONFIG | Create partition
      parted:
        device: '/dev/{{ backup_disk.stdout }}'
        number: 1
        state: present

    - name: BACKUP CONFIG | Create filesystem
      filesystem:
        fstype: ext4
        dev: '/dev/{{ backup_disk.stdout }}1'

    - name: BACKUP CONFIG | Create mount point
      file:
        path: /var/backup
        state: directory

    - name: BACKUP CONFIG | Mount disk
      mount:
        path: /var/backup
        src: '/dev/{{ backup_disk.stdout }}1'
        state: mounted
        fstype: ext4

    - name: BACKUP CONFIG | Add the user 'borg'
      user:
        name: borg
        comment: borg backup

    - name: BACKUP CONFIG | Change file ownership, group and permissions
      file:
        path: /var/backup
        owner: borg
        group: borg

    - name: BACKUP CONFIG | Borg init repo
      shell: rm -rf /var/backup/*
```
Ранее сгенерированный публичный ключ служебного пользователя помещается в список известных на сервере бэкапов:
```yml
    - name: BACKUP CONFIG | fill /home/borg/.ssh/authorized_keys
      authorized_key:
        user: borg
        state: present
        key: "{{ lookup('file', '/tmp/id_rsa.pub/client/root/.ssh/id_rsa.pub') }}"
```
На клиенте инициализируется репозиторий, в который будет выполняться резервное копирование:
```yml
- name: CLIENT CONFIG | Borg
  hosts: client
  become: true
  tasks:
    - name: CLIENT CONFIG | Borg init repo
      shell: borg init --encryption=repokey borg@192.168.11.160:/var/backup/
      environment:
        BORG_NEW_PASSPHRASE: "secret"
```
Подготовлены юниты сервиса для systemd, последовательно выполняется бэкап, проверка бэкапа и удаление старых бэкапов в соответствии с политикой храненения:
```ini
[Unit]
Description="Borg Backup"
[Service]
Type=oneshot
Environment="BORG_PASSPHRASE=secret"
Environment="REPO=borg@192.168.11.160:/var/backup/"
Environment="BACKUP_TARGET=/etc"
ExecStart=/bin/borg create --stats ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}
ExecStart=/bin/borg check ${REPO}
ExecStart=/bin/borg prune --keep-daily 90 --keep-monthly 12 --keep-yearly 1 ${REPO}
```
и таймера, запуск бэкапа выполняется каждые 5 минут:
```ini
[Unit]
Description=Borg Backup
Requires=borg-backup.service
[Timer]
OnUnitActiveSec=5min
[Install]
WantedBy=timers.target
```
Таймер и сервис запущены:
```yml
    - name: CLIENT CONFIG | Borg service & timer
      copy:
        src: '{{ item }}'
        dest: /etc/systemd/system/
      loop:
        - 'borg-backup.service'
        - 'borg-backup.timer'

    - name: CLIENT CONFIG | Borg service & timer
      systemd:
        name: borg-backup.timer
        enabled: yes
        state: started
        daemon_reload: yes
```
Статус таймера:
```sh
[root@client rsyslog.d]# systemctl list-timers
NEXT                         LEFT      LAST                         PASSED   UNIT                         ACTIVATES
Sun 2022-10-02 16:42:04 UTC  291ms ago Sun 2022-10-02 16:42:04 UTC  14ms ago borg-backup.timer            borg-backup.service
Mon 2022-10-03 09:47:54 UTC  17h left  Sun 2022-10-02 09:47:54 UTC  6h ago   systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service

2 timers listed.
Pass --all to see loaded but inactive timers, too.
```
Бэкап выполняется:
```sh
[root@client rsyslog.d]# borg list borg@192.168.11.160:/var/backup/
etc-2022-10-02_16:42:05              Sun, 2022-10-02 16:42:07 [de76a1ffc5c93da9e3e265aac29b4def31b0e71e16ef05374fd507edc95bc7b1]
```

Содержимое бэкапа:
```sh
[root@client rsyslog.d]# borg list borg@192.168.11.160:/var/backup/::etc-2022-10-02_16:42:05
drwxr-xr-x root   root          0 Sun, 2022-10-02 14:08:08 etc
-rw------- root   root          0 Thu, 2020-04-30 22:04:55 etc/crypttab
lrwxrwxrwx root   root         17 Thu, 2020-04-30 22:04:55 etc/mtab -> /proc/self/mounts
-rw-r--r-- root   root      12288 Sun, 2022-10-02 09:32:20 etc/aliases.db
-rw-r--r-- root   root       2388 Thu, 2020-04-30 22:08:36 etc/libuser.conf
-rw-r--r-- root   root       2043 Thu, 2020-04-30 22:08:36 etc/login.defs
-rw-r--r-- root   root         37 Thu, 2020-04-30 22:08:36 etc/vconsole.conf
lrwxrwxrwx root   root         25 Thu, 2020-04-30 22:08:36 etc/localtime -> ../usr/share/zoneinfo/UTC
-rw-r--r-- root   root         19 Thu, 2020-04-30 22:08:36 etc/locale.conf
-rw-r--r-- root   root        450 Sun, 2022-10-02 09:32:32 etc/fstab

...

drwxr-x--- root   root          0 Sun, 2022-10-02 09:32:15 etc/audit
drwxr-x--- root   root          0 Thu, 2020-04-30 22:07:01 etc/audit/rules.d
-rw------- root   root        163 Thu, 2020-04-30 22:07:01 etc/audit/rules.d/audit.rules
-rw-r----- root   root         81 Sun, 2022-10-02 09:32:15 etc/audit/audit.rules
-rw-r----- root   root        127 Thu, 2019-08-08 12:06:02 etc/audit/audit-stop.rules
-rw-r----- root   root        805 Thu, 2019-08-08 12:06:02 etc/audit/auditd.conf
drwxr-x--- root   root          0 Thu, 2020-04-30 22:09:26 etc/sudoers.d
-r--r----- root   root         33 Thu, 2020-04-30 22:09:26 etc/sudoers.d/vagrant
```

Журналы содержат события запуска сервиса:
```log
[root@client system]# journalctl
...
Oct 02 16:31:04 client systemd[1]: Starting "Borg Backup"...
Oct 02 16:31:08 client borg[30195]: ------------------------------------------------------------------------------
Oct 02 16:31:08 client borg[30195]: Archive name: etc-2022-10-02_16:31:04
Oct 02 16:31:08 client borg[30195]: Archive fingerprint: dc3376a7e354481c054b846679513357aa5e0b710652280acd52132eaf8fec1a
Oct 02 16:31:08 client borg[30195]: Time (start): Sun, 2022-10-02 16:31:07
Oct 02 16:31:08 client borg[30195]: Time (end):   Sun, 2022-10-02 16:31:08
Oct 02 16:31:08 client borg[30195]: Duration: 0.86 seconds
Oct 02 16:31:08 client borg[30195]: Number of files: 1713
Oct 02 16:31:08 client borg[30195]: Utilization of max. archive size: 0%
Oct 02 16:31:08 client borg[30195]: ------------------------------------------------------------------------------
Oct 02 16:31:08 client borg[30195]: Original size      Compressed size    Deduplicated size
Oct 02 16:31:08 client borg[30195]: This archive:               28.52 MB             13.54 MB                715 B
Oct 02 16:31:08 client borg[30195]: All archives:               57.03 MB             27.07 MB             11.88 MB
Oct 02 16:31:08 client borg[30195]: Unique chunks         Total chunks
Oct 02 16:31:08 client borg[30195]: Chunk index:                    1296                 3426
Oct 02 16:31:08 client borg[30195]: ------------------------------------------------------------------------------
Oct 02 16:31:16 client systemd[1]: Started "Borg Backup".
Oct 02 16:37:04 client systemd[1]: Starting "Borg Backup"...
Oct 02 16:37:08 client borg[30209]: ------------------------------------------------------------------------------
Oct 02 16:37:08 client borg[30209]: Archive name: etc-2022-10-02_16:37:04
Oct 02 16:37:08 client borg[30209]: Archive fingerprint: 63be14ecb4be475a8e7e3bbcf61f9d24f0e6225fdf054da0cb96a83e8e6a25ff
Oct 02 16:37:08 client borg[30209]: Time (start): Sun, 2022-10-02 16:37:07
Oct 02 16:37:08 client borg[30209]: Time (end):   Sun, 2022-10-02 16:37:08
Oct 02 16:37:08 client borg[30209]: Duration: 0.87 seconds
Oct 02 16:37:08 client borg[30209]: Number of files: 1713
Oct 02 16:37:08 client borg[30209]: Utilization of max. archive size: 0%
Oct 02 16:37:08 client borg[30209]: ------------------------------------------------------------------------------
Oct 02 16:37:08 client borg[30209]: Original size      Compressed size    Deduplicated size
Oct 02 16:37:08 client borg[30209]: This archive:               28.52 MB             13.54 MB                715 B
Oct 02 16:37:08 client borg[30209]: All archives:               57.03 MB             27.07 MB             11.88 MB
Oct 02 16:37:08 client borg[30209]: Unique chunks         Total chunks
Oct 02 16:37:08 client borg[30209]: Chunk index:                    1296                 3426
Oct 02 16:37:08 client borg[30209]: ------------------------------------------------------------------------------
Oct 02 16:37:16 client systemd[1]: Started "Borg Backup".
```

# **Результаты**

Полученный в ходе работы `Vagrantfile` помещен в публичный репозиторий:

- **GitHub** - https://github.com/jimidini77/otus-linux-day26
