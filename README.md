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

#Systemdで管理  
#ユニットファイルの作成  
/usr/lib/systemd/system/mongod.service
```
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network-online.target
Wants=network-online.target

[Service]
User=mongod
Group=mongod
Environment="OPTIONS=-f /etc/mongod.conf"
Environment="MONGODB_CONFIG_OVERRIDE_NOFORK=1"
Environment="GLIBC_TUNABLES=glibc.pthread.rseq=0"
EnvironmentFile=-/etc/sysconfig/mongod
ExecStart=/usr/bin/mongod $OPTIONS
RuntimeDirectory=mongodb
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false
# Recommended limits for mongod as specified in
# https://docs.mongodb.com/manual/reference/ulimit/#recommended-ulimit-settings

[Install]
WantedBy=multi-user.target
```
#設定ファイルのロード
```
systemctl deamon-reload
```

#起動
```
$ systemctl start mongod
```
#停止
```
$ systemctl stop mongod
```


# MongoDBの初期設定  
#スーパーユーザ作成  
```
# mongosh
> use admin
> db.createUser(
    {
        user: "admin",
        pwd: "tkhA5119083?",
        roles: [ { role: "root", db: "admin" } ]
    }
)
```
#認証の有効化  

```
$ mongosh
```
※ユーザを作成後に人称の有効化をしないとログインできなくなってしまうので注意！  
```
$ sudo vi /etc/mongod.conf
```
以下の内容を追加
```
security:
  authorization: enabled
```

#MongoDBの再起動
```
# systemctl restart mongod
```

#バインドIPの変更
```
$ sudo vi /etc/mongod.conf
```
以下の内容を追加
```
net:
  port: 27017
  bindIp: 0.0.0.0 
```

#接続
```
$ mongosh -u admin -p
```

# データベースとコレクションの操作

#データベースの作成
```
> use testdb
```

#データベースの削除  
※現在接続されているデータベースが削除される
```
> db.dropDatabase()
```

#データベース管理者を作成
```
> db.createUser(
    {
        user: "testdbOwner",
        pwd: "tkhA5119083?",
        roles: [ { role: 'dbOwner', db: "testdb" } ]
    }
)
```

#ユーザの削除
```
> db.dropUser("testdbOwner")
```

#ユーザのパスワード変更
```
> db.changeUserPassword("testdbOwner", "password")
```

#コレクション作成
```
> db.createCollection("testcollection")
```

#コレクションの作成
```
> db.testdb.drop()
```

#キャップド・コレクション作成  
//最大1メガバイト
```
> db.createCollection( "logSize", { capped: true, size: 1048576 } )
```

//最大1万件
```
> db.createCollection( "logMax", { capped: true, max: 10000 } )
```

# ドキュメントの操作

//1件のドキュメントを追加
```
> db.fishes.insert( { name: "マグロ", price: 1000 } )
```

//複数件のドキュメントを追加
```
> db.fishes.insert([
    {name: "サンマ", price:360},
    {name: "タイ", price:500},
    {name: "サバ", price:400}
])
```

//全件検索
```
> db.fishes.find()
```

//_idを除外して全件検索
```
> db.fishes.find(
    {},
    { _id: 0 }
)
```

//完全一致検索
```
> db.fishes.find(
    { name:"マグロ"},
    {_id: 0 }
)
```

//部分一致検索
```
> db.fishes.find(
    { name:/サ/ },
    { _id: 0 }
)
```

//範囲検索例1
```
> db.fishes.find(
    { "price": { $lt: 500 } },
    { _id: 0 }
)
```

//範囲検索例2
```
> db.fishes.find(
    { "price": { $gte: 300, $lte: 500}},
    { _id: 0}
)
```

//範囲検索例3
```
> db.fishes.find(
    { $and: [
        { "name": /サ/},
        { "price": { $gte: 400 } }
    ]},
    { _id: 0 }
)
```

//ドキュメント更新
```
> db.fishes.update(
    { "name": "マグロ" },
    { $set: { "price": 2000 } }
)
```

//フィールドの追加  
存在しないフィールドを更新すると、エラーとはならず素のフィールドが追加される
```
> db.fishes.update(
    { },
    { $set: { "part" : null } },
    { multi: true }
)
```

//フィールドの削除
```
> db.fishes.update(
    { },
    { $unset: { "part": "" } },
    { multi: true }
)
```

//UPSERT
```
> db.fishes.update(
    { "name": "カツオ"},
    { $set: { "price": 450 } },
    { upsert: true }
)
```

//ドキュメントを1件削除
```
> db.fishes.deleteOne(
    { name: "カツオ"}
)
```

//ドキュメントを複数件削除
```
> db.fishes.deleteMany(
    { name: /サ/ }
)
```

//インデックス作成
```
> db.fishes.createIndex(
     { name: 1 }
)
```

//インデックス削除  
インデックス名は上記のコマンドで自動で生成されている。オプションで設定も可能。  
db.コレクション名.getIndexes()で確認可能
```
> db.fishes.dropIndex("インデックス名")
```


# レプリカセット  

#レプリカセット構築手順  
冗長化という点では意味がないが、テスト的に同じサーバ内にMongoDBインスタンスを複数起動する方法で説明
```
//データベースディレクトリの作成
$ sudo mkdir /var/lib/mongo-sec
$ sudo mkdir /var/lib/mongo-arb

//ログディレクトリの作成
$ sudo mkdir /var/log/mongodb-sec
$ sudo mkdir /var/log/mongodb-arb

//所有者の変更
$ sudo chown mongod:mongod /var/lib/mongo-sec
$ sudo chown mongod:mongod /var/lib/mongo-arb
$ sudo chown mongod:mongod /var/log/mongodb-sec
$ sudo chown mongod:mongod /var/log/mongodb-arb

//設定ファイルを複製
$ sudo cp -p /etc/mongod.conf /etc/mongod-sec.conf
$ sudo cp -p /etc/mongod.conf /etc/mongod-arb.conf
```

//設定ファイルを編集  
mongod-sec.conf
```
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb-sec/mongod.log

# Where and how to store data.
storage:
  dbPath: /var/lib/mongo-sec

# how the process runs
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod-sec.pid
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27018
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.


# security:
#   authorization: enabled

#operationProfiling:

#replication:

#sharding:

## Enterprise-Only Options

#auditLog:

```

mongod-arb.conf
```
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb-arb/mongod.log

# Where and how to store data.
storage:
  dbPath: /var/lib/mongo-arb

# how the process runs
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod-arb.pid
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27019
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.


# security:
#   authorization: enabled

#operationProfiling:

#replication:

#sharding:

## Enterprise-Only Options

#auditLog:

```

//セカンダリとアービターの起動  
systemd経由でmongodを起動していないと、/var/run/mongodbにPIDファイルを生成できない。
```
$ sudo -u mongod /usr/bin/mongod -f /etc/mongod-sec.conf
$ sudo -u mongod /usr/bin/mongod -f /etc/mongod-arb.conf
```

//セカンダリとアービターへの接続
```
$ mongosh --port 27018
$ mongosh --port 27019
```
スーパーユーザ作成と認証の有効化を行う。認証の有効化。  

//セカンダリとアービターへのスーパーユーザでの接続
```
$ mongosh --port 27018 -u admin -p
$ mongosh --port 27019 -u admin -p
```

//レプリカセットにキーファイルを作成
レプリカセット同士はキーファイルという全く同じ内容のファイルを持つことで正しいメンバーであることを確認する
```
#キーファイルの中身はBASE64セットで使用できる文字のみをつかった6バイトから1024バイトのテキストファイルであるひつようがある。
#opensslコマンドを使用して、ランダムな文字列を出力する
$ sudo openssl rand -base64 756 > /tmp/keyFile
$ sudo mv /tmp/keyFile /etc/keyFile
#キーファイルは所有者以外にアクセス権限があってはいけない。
$ sudo chown mongod:mongod /etc/keyFile
$ sudo chmod 400 /etc/keyFile
```
本来は各サーバごとにキーファイルをSCPでコピーする必要があるが、今回の同じサーバの異なるインスタンスでレプリカセットにする場合は不要。

//すべての設定ファイルに以下を追加する。
```
security:
  authorization: enabled
  keyFile: /etc/keyFile

replication:
  oplogSizeMB: 1024          ←oplogの最大サイズをMB単位で指定
  replSetName: replSet01     ←任意のレプリカセット名
```

セカンダリの停止
```
$ sudo mongod -f /etc/mongod-sec.conf --shutdown
$ sudo mongod -f /etc/mongod-arb.conf --shutdown
```

再起動
```
$ systemctl restart mongod
$ sudo -u mongod /usr/bin/mongod -f /etc/mongod-sec.conf
$ sudo -u mongod /usr/bin/mongod -f /etc/mongod-arb.conf
```

もしも起動ができない場合は、SELinuxを一時的にPermissiveに切り替えるなどする
```
$ sudo getenforce
$ sudo setenforce 0
```

//レプリカセットの構築  
どのインスタンスでも問題ないが、一応プライマリにしたいインスタンスにログインする
```
$ mongosh --port 27017 -u admin -p
> rs.initiate({
    _id : "replSet01",
    members: [
        { _id : 0, host : "localhost:27017", priority: 10 },
        { _id : 1, host : "localhost:27018", priority: 1},
        { _id : 2, host : "localhost:27019", arbiterOnly: true}
    ]
})
```

//自動フェールオーバーをためしてみる  
//プライマリの停止
```
$ systemctl stop mongod
```

//セカンダリにログイン  
プライマリになっていることが確認できる。
```
$ mongosh --port 27018 -u admin -p
> rs.status()
```
確認後に再び停止したものを起動してあげると、レプリカセットに復帰したことも確認できる。  

# MongoDBのシャーディング構成

MongoDBをすべて停止する  
※運用後にシャーディングする場合はMongoDBをすべて停止する必要があるので本当にシャーディングする必要があるのかよく考えてから決定する
```
$ sudo mongod -f /etc/mongod-sec.conf --shutdown
$ sudo mongod -f /etc/mongod-arb.conf --shutdown
$ systemctl stop mongod
```

セカンダリとプライマリで設定ファイルの編集をおこなう
/etc/mongod.conf
/etc/mongod-sec.conf
```
# security:                     ←コメントアウト
#   authorization: enabled      ←コメントアウト
#   keyFile: /etc/keyFile       ←コメントアウト

sharding:                       ←追加    
  clusterRole: shardsvr         ←追加

```

アービターはセキュリティ設定だけを無効にする
/etc/mongod-arb.conf
```
# security:                     ←コメントアウト
#   authorization: enabled      ←コメントアウト
#   keyFile: /etc/keyFile       ←コメントアウト
```

設定ファイルを編集したら、MongoDBを再起動して、シャーディングするデータベースとコレクションを作成する
```
$ systemctl restart mongod
$ sudo -u mongod /usr/bin/mongod -f /etc/mongod-sec.conf
$ sudo -u mongod /usr/bin/mongod -f /etc/mongod-arb.conf
```

プライマリにログインし、競争馬の名前とレースを記録するコレクションを作成
```
$ mongosh --port 27017
> use horse
> db.createCollection("racehorse")
> db.racehorse.insert([
  { "name": "タイキシャトル", "race":{ "date": "1998-11-22", "raceName": "マイルチャンピオンシップ"} },
  { "name": "タイキシャトル", "race":{ "date": "1998-06-04", "raceName": "安田記念"} },
  { "name": "タイキシャトル", "race":{ "date": "1997-12-14", "raceName": "スプリンターズステークス"} },
  { "name": "トウカイテイオー", "race":{ "date": "1993-12-26", "raceName": "有馬記念"} },
  { "name": "トウカイテイオー", "race":{ "date": "1992-11-29", "raceName": "ジャパンカップ"} },
  { "name": "トウカイテイオー", "race":{ "date": "1991-05-26", "raceName": "東京優駿"} }
])
//シャードキーにはインデックスが必要
> db.racehorse.createIndex({ name: 1})
```