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
※ユーザを作成後に人称の有効化をしないとログインできなくなってしまうので注意！  
```
# sudo vi /etc/mongod.conf
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
sudo vi /etc/mongod.conf
```
以下の内容を追加
```
net:
  port: 27017
  bindIp: 0.0.0.0 
```

#接続
```
# mongosh -u admin -p
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