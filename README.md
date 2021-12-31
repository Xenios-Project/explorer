XNC Explorer - 1.7.4
================

Fork of the open source iquidus block explorer written in node.js. 

**Note** First sync needs to complete for the top100 to work.

### Requirements

*  node.js >= 8.17.0 (12.14.0 is advised for updated dependencies)
*  mongodb 4.2.x
*  *coind

### Setup
#### Install Requirements (ubuntu 18LTS)
```
wget -qO - https://www.mongodb.org/static/pgp/server-4.2.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.2.list
sudo apt-get update
sudo apt install nodejs mongodb-org -y
sudo systemctl stop mongod.service
sudo systemctl start mongod.service
sudo systemctl enable mongod.service
```

#### Create database

Generate secure password:

    > db_password=$(cat /dev/urandom | tr -dc a-zA-Z0-9%^@\!$ | fold -w 36 | head -n 1)

Enter MongoDB cli:

    $ mongo

Create databse/user:

    echo -e 'use explorerdb' > /tmp/mongo.js
    echo -e "db.createUser( { user: \"iquidus\", pwd: \"${db_password}\", roles: [ \"readWrite\" ] } )" >> /tmp/mongo.js
    mongo < /tmp/mongo.js

#### Get the source

    git clone https://github.com/Xenios-Project/explorer explorer

#### Install node modules

    cd explorer && npm install --production

### Configure
Configure settings.json

*This should cover it, but make required changes in settings.json*
```
cp ./settings.json.template ./settings.json
xncpass=$(grep -Po '(?<=rpcpassword=).*' ~/.xenios/xenios.conf)
rpcuser=$(grep -Po '(?<=rpcuser=).*' ~/.xenios/xenios.conf)
sed "s/wallet_user_placeholder/$rpcuser/" -i ./settings.json
sed "s/wallet_password_placeholder/$xncpass/" -i ./settings.json
sed "s/mongo_password_placeholder/$db_password/" -i ./settings.json
```

Setup crontab (should be run from explorer root directory. Otherwise edit the paths)
```
sudo -s -- <<EOF
(crontab -l 2>/dev/null; echo "*/1 * * * * cd $(pwd) && /usr/bin/nodejs scripts/sync.js index update > /dev/null 2>&1") | crontab -
(crontab -l 2>/dev/null; echo "*/2 * * * * cd $(pwd) && /usr/bin/nodejs scripts/sync.js market > /dev/null 2>&1") | crontab -
(crontab -l 2>/dev/null; echo "*/5 * * * * cd $(pwd) && /usr/bin/nodejs scripts/peers.js > /dev/null 2>&1") | crontab -
EOF
```

### Start Explorer

    screen -S explorer
    (npm start &)

*Note: mongod must be running to start the explorer*

As of version 1.4.0 the explorer defaults to cluster mode, forking an instance of its process to each cpu core. This results in increased performance and stability. Load balancing gets automatically taken care of and any instances that for some reason die, will be restarted automatically. For testing/development (or if you just wish to) a single instance can be launched with

    node --stack-size=10000 bin/instance

To stop the cluster you can use

    npm stop

### Syncing databases with the blockchain

sync.js (located in scripts/) is used for updating the local databases. This script must be called from the explorers root directory.

    Usage:   [database] [mode]

    database: (required)
    index [mode] Main index: coin info/stats, transactions & addresses
    market       Market data: summaries, orderbooks, trade history & chartdata

    mode: (required for index database only)
    update       Updates index from last sync to current block
    check        checks index for (and adds) any missing transactions/addresses
    reindex      Clears index then resyncs from genesis to current block

    notes:
    * 'current block' is the latest created block when script is executed.
    * The market database only supports (& defaults to) reindex mode.
    * If check mode finds missing data(ignoring new data since last sync),
      index_timeout in settings.json is set too low.


*It is recommended to have this script launched via a cronjob at 1+ min intervals.*

**crontab**

*Example crontab; update index every minute and market data every 2 minutes*

    */1 * * * * cd /path/to/explorer && /usr/bin/nodejs scripts/sync.js index update > /dev/null 2>&1
    */2 * * * * cd /path/to/explorer && /usr/bin/nodejs scripts/sync.js market > /dev/null 2>&1
    */5 * * * * cd /path/to/explorer && /usr/bin/nodejs scripts/peers.js > /dev/null 2>&1

### Wallet

`xeniosd` must be running.
    
### Security

Ensure mongodb is not exposed to the outside world via your mongo config or a firewall to prevent outside tampering of the indexed chain data. 

### Known Issues

**script is already running.**

If you receive this message when launching the sync script either a) a sync is currently in progress, or b) a previous sync was killed before it completed. If you are certian a sync is not in progress remove the index.pid and db_index.pid from the tmp folder in the explorer root directory.

    rm tmp/*

**exceeding stack size**

    RangeError: Maximum call stack size exceeded

Nodes default stack size may be too small to index addresses with many tx's. If you experience the above error while running sync.js the stack size needs to be increased.

To determine the default setting run

    node --v8-options | grep -B0 -A1 stack_size

To run sync.js with a larger stack size launch with

    node --stack-size=[SIZE] scripts/sync.js index update

Where [SIZE] is an integer higher than the default.

*note: SIZE will depend on which blockchain you are using, you may need to play around a bit to find an optimal setting*

### License

Copyright (c) 2015, Iquidus Technology  
Copyright (c) 2015, Luke Williams  
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice,
  this list of conditions and the following disclaimer in the documentation
  and/or other materials provided with the distribution.

* Neither the name of Iquidus Technology nor the names of its
  contributors may be used to endorse or promote products derived from
  this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
