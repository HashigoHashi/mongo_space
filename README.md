# MongoDBのインストールと起動停止  
#ネットワーク  
～ファイアウォールとSELinuxでMongoDBが使用するポート(27017)が閉じられて異なことを確認～  

#ファイアウォール
```
sudo firewall-cmd --state
sudo firewall-cmd --list-ports
```
#SELinux
```
sestatus
sudo firewall-cmd --list-ports
```
#Transparent Huge Pages 無効化
#BLSエントリファイルを確認
```
[takahashi_daigo@localhost ~]$ sudo ls /boot/loader/entries/
f4d751f805ee46a0913f76b9bb80739a-0-rescue.conf			    f4d751f805ee46a0913f76b9bb80739a-5.14.0-570.30.1.el9_6.x86_64.conf
f4d751f805ee46a0913f76b9bb80739a-5.14.0-570.25.1.el9_6.x86_64.conf  f4d751f805ee46a0913f76b9bb80739a-5.14.0-570.33.2.el9_6.x86_64.conf
```
#現在使用中のカーネルを確認
```
[takahashi_daigo@localhost ~]$ uname -r
5.14.0-570.33.2.el9_6.x86_64
```
#対象のBLSファイルを編集
#options行にtransparent_hugepage=neverを追加
```
[takahashi_daigo@localhost ~]$ sudo vi /boot/loader/entries/f4d751f805ee46a0913f76b9bb80739a-5.14.0-570.33.2.el9_6.x86_64.conf
```
#再起動
```
[takahashi_daigo@localhost ~]$ sudo reboot
```
#確認
```
[takahashi_daigo@localhost ~]$ cat /sys/kernel/mm/transparent_hugepage/enabled 
always madvise [never]
```

#ulimit設定変更  
https://www.mongodb.com/ja-jp/docs/manual/reference/ulimit/#recommended-ulimit-settings  
#確認  
```
ulimit -a
```
#yumでインストール  
https://www.mongodb.com/ja-jp/docs/manual/tutorial/install-mongodb-on-amazon/#install-mongodb-community-edition  
#リポジトリファイルを作成  
```
[takahashi_daigo@localhost ~]$ cd /etc/yum.repos.d/
[takahashi_daigo@localhost yum.repos.d]$ sudo touch mongodb-org-8.0.repo
```
以下の内容で用意
```
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2023/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-8.0.asc
```
#MongoDBのインストール
```
sudo yum install -y mongodb-org
```


#起動準備
#MongoDB管理ユーザの作成
```
sudo useradd -s /bin/false mongod
```
| 用途 | ディレクトリーパス |
|--------------|------------------------|
| データ  | /var/lib/mongo        |
| ログファイル  | /var/log/mongodb  |
| PIDファイル | /var/run/mongodb              |

#ディレクトリはMongoDB管理ユーザを所有者にする
```
sudo chown -R mongod:mongod /var/lib/mongo
sudo chown -R mongod:mongod /var/log/mongodb
sudo chown -R mongod:mongod /var/run/mongodb
```

#設定ファイル作成
```
sudo vi /etc/mongod.conf
```
以下の内容で用意
```
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

storage:
  dbPath: /var/lib/mongo

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 27017
  bindIp: 127.0.0.1 
```

#起動
#child process started successfully, parent exitingと表示されれば起動完了
```
sudo -u mongod /usr/bin/mongod -f /etc/mongod.conf
```

#停止
```
sudo -u mongod mongod -f /etc/mongod.conf --shutdown
```