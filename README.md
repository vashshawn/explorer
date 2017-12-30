# Litedoge explorer installation guide

Copy-paste commands to run Iquidus explorer for Litedoge on Ubuntu.

## Prerequisites

`litedoged`

Explorer needs a local `litedoged` node to run, fully synced, configured to listen rpc calls, and `txindex` set to true. 

Before installing, follow the litedoge full-node copy-paste installation guide at https://github.com/suchapp/litedoge to setup litedoge full node with correct settings.

`mongodb`

Follow steps to install mongo-db at https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/ or copy-paste these commands:

```
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2930ADAE8CAF5059EE73BB4B58712A2291FA4AD5

echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.6.list

apt update

apt install -y mongodb-org

systemctl enable mongod.service

service mongod start
```

## Create database and mongodb user

Open mongo shell:

```
mongo
```

Edit mongodb password before copy-pasting

```
use litedogeexplorerdb
db.createUser( { user: "litedogeexplorer", pwd: "mongodb-password-here", roles: [ "readWrite" ] } )
```

## Create user

You don't want start litedoge explorer with the root user. Create user `explorer` without log-in rights.

```
sudo useradd -s /usr/sbin/nologin -m explorer
```

## Clone the explorer repository

```
cd /home/explorer && git clone https://github.com/iquidus/explorer

npm install --production
```

Note: Command above clones the original explorer. If you want to use litedoge theme, you'll need to clone this repository instead.

## Configure

Edit mongodb password and rpc-password before copy-pasting

```
cat << "EOF" | sudo tee /home/explorer/explorer/settings.json

{
  "title": "LDOGE ranking",
  "address": "127.0.0.1:3001",
  "port" : 3001,
  "coin": "LiteDoge",
  "symbol": "LDOGE",
  "logo": "/images/logo.png",
  "favicon": "public/favicon.ico",
  "theme": "Superhero", // Uses bootswatch themes (http://bootswatch.com/)

  // LiteDoge chain settings
  "genesis_tx": "0000032101032f27e7cdddb1196353f7fc9e1b6294717432135add95534f67c6",
  "genesis_block": "b2926a56ca64e0cd2430347e383f63ad7092f406088b9b86d6d68c2a34baef51",
  "txcount": 100, //amount of txs to index per address (stores latest n txs)
  "supply": "GETINFO",
  "confirmations": 40, // confirmations
  "nethash": "netmhashps",

  // database settings (MongoDB)
  "dbsettings": {
    "user": "litedogeexplorer",
    "password": "mongodb-password-here",
    "database": "litedogeexplorerdb",
    "address": "127.0.0.1",
    "port": 27017
  },

  // wallet settings
  "wallet": {
    "host": "localhost",
    "port": 9332,
    "user": "litedogerpc",
    "pass": "rpc-password-here"
  },

  "update_timeout": 10, // update script settings
  "check_timeout": 250,

  "locale": "locale/en.json", // language settings

  "display": { // menu settings
    "api": false,
    "richlist": true,
    "search": true,
    "markets": true
  },

  "markets": {
    "coin": "LDOGE",
    "exchange": "LTC",
    "enabled": ["cryptopia"],
    "cryptopia_id": "1818",
    "ccex_key" : "",
    "default": "cryptopia"
  },

  "index": {
    "show_hashrate": true,
    "difficulty": "Hybrid",
    "last_txs": 100
  },

  "richlist": { // top100 settings
    "distribution": false,
    "received": false,
    "balance": true
  }
}

EOF
```

## Chmod

```
sudo chown -R explorer:explorer /home/explorer/explorer && cd /home/explorer/explorer
```

## Create `run-explorer` run script for daemon

```
cat << "EOF" | sudo tee /bin/run-explorer

#!/bin/sh -

/usr/bin/node --stack-size=10000 /home/explorer/explorer/bin/cluster

EOF
```

Chmod 

```
chmod a+x /bin/run-explorer
```

## Create systemd service to start explorer automatically

We want our explorer to start automatically after a reboot or crash. Create a systemd service for it.

```
cat << "EOF" | sudo tee /lib/systemd/system/explorer.service
[Unit]
Description=Litedoge explorer
After=network.target

[Service]
Type=simple
User=explorer
ExecStart=/usr/bin/node --stack-size=10000 /home/explorer/explorer/bin/cluster
WorkingDirectory=/home/explorer/explorer
Restart=on-failure
SyslogIdentifier=LDOGEXPLR
RestartSec=30

StandardOutput=syslog
StandardError=syslog

[Install]
WantedBy=multi-user.target
Alias=explorer.service

EOF
```

Enable the service:

```
systemctl daemon-reload && systemctl enable explorer.service
```

Every time you change something in `/lib/systemd/system/explorer.service`, you need to run `sudo systemctl daemon-reload`.

## Start the service

```
service explorer start
```

Since we have created an alias for the service, we will be able to work with it in the future as follows:

```
sudo service explorer start|stop|restart|status
```

## Start the initial sync and create missing indexes

Both `litedoged` and `explorer` needs to be running for sync to work.

```
cd /home/explorer/explorer

node scripts/sync.js index reindex
```

Open mongo shell:

```
mongo
```

Create missing indexes:

```
use litedogeexplorerdb

db.addresses.createIndex({"a_id":1})

db.addresses.createIndex({"timestamp":1})

db.txes.createIndex({"txid":1})

db.txes.createIndex({"timestamp":1})

db.txes.createIndex({"blockindex":1})

```

## Restart sync if needed

If sync fails or you need to interrupt it, remove the .pid file and restart sync with `update` parameter:

```
rm tmp/index.pid

node scripts/sync.js index update
```

## Set crontab to update db every minute

```
crontab -e

*/1 * * * * cd /home/explorer/explorer && /usr/bin/nodejs scripts/sync.js index update > /dev/null 2>&1
*/2 * * * * cd /home/explorer/explorer && /usr/bin/nodejs scripts/sync.js market > /dev/null 2>&1
*/5 * * * * cd /home/explorer/explorer && /usr/bin/nodejs scripts/peers.js > /dev/null 2>&1
```
