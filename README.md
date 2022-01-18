#Ansible 的組態檔案

[root@IT-UAT-020242 kai]# cat ansible.cfg
[defaults]                  # 預設值
inventory = ./inventory
remote_user = root
ask_pass = false

[privilege_escalation]      # 特權升級
become = false
#become_method = sudo
become_user = root
become_ask_pass = false


#inventory 清單列表

[root@IT-UAT-020242 kai]# cat inventory
[managed]
172.16.20.243

inventory 內部的主機名稱設計，大致上可以分為底下這幾類：
# 1. 單一主機，沒有分類的效果，其實是放在 ungrouped 群組當中
gitclient
gitclient[01:20].ksu    <==代表 gitclient01.ksu, gitclient02.ksu... gitclient20.ksu
172.17.20.[1:30]        <==代表 172.17.20.1, 172.17.20.2... 172.17.20.30

# 2. 歸類於 website 的群組與 devel 群組
[website]
webserver1
dbserver1

[devel]
webserver1
devnode1
pronode1

# 3. 成立一個『父群組』，名稱為 mylab，其中 mylab 的子群組包含 devel 與 website：
[mylab:children]
website
devel


檢查 managed 群組內有幾部受管理的主機名稱
ansible managed -i /root/kai/inventory --list-hosts

檢查一下系統管理的所有主機
ansible all -i inventory --list-hosts   (通常建議自行修改清單檔案，如果要使用清單檔案， 可以透過 ansible -i inventory_file 來處理)
ansible all  --list-hosts (預設的 ansible inventory 檔名為 /etc/ansible/hosts)

檢查一下，沒有分到其他類別，而是各自存在的主機名稱？
ansible ungrouped -i inventory --list-hosts

兩個預設存在的群組：
all：代表所有的主機
ungrouped：代表沒有分類的主機

====使用模組與 ad hoc 指令模式===
詢目前 ansible 的所有模組以及特殊模組名稱：
$ ansible-doc -l
查詢關於 copy 相關的模組：
$ ansible-doc -l | grep copy
ansible-doc copy

基本指令語法如下：
$ ansible [host|group] -m module [-a 'module args']
也就是說，透過某個模組的功能，讓某部主機或某群組主機，進行某個設定。而模組也可能會有額外的參數， 那就是搭配 -a '模組參數' 來處理了。
ansible managed  -m copy -a 'src=/etc/hosts dest=/etc/hosts'

##提供給管理員一個 debug 的功能！管理員不需要直接登入 managed host， 直接以 command 模組，就能夠處理 managed host 的內部指令操作了。
基本語法如下：
$ ansible [host|group] -m command -a '在遠端主機執行的指令'
ansible managed -m command -a 'id root'

##基本上， command 模組一次只執行一個指令，其他的資料都被當成是該指令的參數，所以， 使用 command 模組時，是不能使用常見的 shell 功能的！如果要使用 shell 功能，或者是要使用 shell 內建的指令， 那就得要使用 shell 模組了。
shell 模組功能：
shell 模組可以取代 command 模組，可以運作的功能比較強大！例如上面提到的連續指令下達：
$ ansible webserver1 -m shell -a 'id demouser; cat /etc/hosts'

檔案相關模組：
copy ：從 control node 複製資料到 managed host
file ：設定檔案權限或其他參數
lineinfile ：判斷搜尋某一個資料行是否在或不在某個檔案中
synchronize： 使用 rsync 進行資料同步
軟體安裝與移除等相關模組：
package： 依據 distribution 的版本，自動判斷需要的軟體管理指令
yum： 使用 yum
apt： 使用 apt-get 等軟體功能 (debian/ubuntu)
dnf： 使用 dnf (fedora/redhat)
pip： 透過 PyPI 管理 python 的可安裝模組
系統管理模組：
firewalld： 使用 firewalld 管理 managed host 防火牆
reboot： 對 managed host 重新開機
service： 對服務管理
user： 新增/移除/管理使用者帳號與密碼
網路工具模組：
get_url： 從 http, https, ftp 等網站下載檔案資料
nmcli： 就是管理網路參數設定的模組
uri：與網路服務互動


====playbook====
#可以直接使用 [tab] 來達成對齊之外，編輯的時候， .yml 格式也會自動縮排對齊！
vim ~/.vimrc
autocmd FileType yaml setlocal ai ts=2 sw=2 et

1.
[root@IT-UAT-020242 kai]# cat copy_hosts.yml
---
  - name: copy /etc/hosts
    hosts: managed
    tasks: 
    - name: copy /etc/hosts to managed /etc/hosts
      copy:
        src: /etc/hosts
        dest: /etc/hosts

2.
[root@IT-UAT-020242 kai]# cat user_add.yml
---
  - name: adduser myuser1
    hosts: managed
    tasks: 
    - name: user:myuser1_uid:3101_passwd:123456
      user:
        name: demouser 
        uid: 3000
        password: "{{'123456'|password_hash('sha512')}}"
