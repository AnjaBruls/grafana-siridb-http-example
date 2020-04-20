# Using Grafana with SiriDB

The goal of this blog is to setup a Grafana dashboard using the SiriDB plugin. For an attractive dashboard we need some data which we can visualize.
In this tutorial we use a Python script which collects some cpu, disp and memory data from the localhost and some imformation about the running
SiriDB processes. All this data will be stored in a SiriDB database and by using a Grafana Dashboard we are able to monitor the data.

We use a fresh Raspian stretch lite installation.
To be precise: Rasphian stretch lite minimal image, version:September 2017

```
sudo apt update
sudo apt upgrade
```
Our `apt upgrade` prompted a reboot was required so we did.

```
sudo reboot
```
Continue the install by installing the prerequisites
```
sudo apt install libuv1 git python3-pip libpcre3-dev libuv1-dev uuid-dev
```
One of the prerequisites when compiling SiriDB from source is libcleri.
```
cd /tmp
git clone https://github.com/transceptor-technology/libcleri.git
cd libcleri/Release
make all
sudo make install
```

After successful install we can continue installing SiriDB siridb-server

```
cd /tmp
git clone https://github.com/transceptor-technology/siridb-server.git
cd siridb-server/Release
make clean
make
sudo cp siridb-server /usr/local/bin/siridb-server
```

Since version 2.0.35 it is possible to use the HTTP API to create and manage databases. However it is also possible to use the [admin tool](https://github.com/SiriDB/siridb-admin) for this.

There are several native clients available for communicating with SiriDB but for Grafana we will use the HTTP API.

SiriDB can scale data across multiple pools and each pool can have two servers for redundancy. We can play with this
concept on a single host by running SiriDB multiple times using different ports. In a real scenario you should use
different nodes but for now we will create four SiriDB nodes and setup two pools, each with two SiriDB servers.

This will create four SiriDB configuration files:
```
for i in {0..3}; do `cat <<EOT > siridb$i.conf
[siridb]
listen_client_port = 900$i
server_name = %HOSTNAME:901$i
ip_support = ALL
optimize_interval = 900
heartbeat_interval = 30
default_db_path = ./dbpath$i
max_open_files = 512
http_api_port = 902$i
EOT` && mkdir dbpath$i; done
```

We can start the SiriDB servers! The following command starts the four SiriDB servers in the background.
```
for i in {0..3}; do siridb-server -c siridb$i.conf > siridb$i.log & done
```
> Hint: you can view the output from a SiriDB process by using for example `cat siridb0.log` or `tail -f siridb0.log`.

Now we use the SiriDB HTTP API to create the actual database. SiriDB has a default service account `sa` with password `siri` which we will use.
For our tutorial we only need a database with `second` precision. We also select a shard duration of 6 hours for this database
because our measurement interval will be only a few seconds. Sometimes you might want to store one value per measurement in each hour or even per day
in which case your database will perform better by using a larger shard duration.

Create the database on the first SiriDB server which is running on port `9000` using curl with basic authentication:

```
curl --location --request POST 'http://localhost:9020/new-database' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic c2E6c2lyaQ==' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "dbname": "tutorialdb",
    "time_precision": "s",
    "buffer_size": 8192,
    "duration_num": "6h",
    "duration_log": "3d"
}'
```

Now we have a database and we can use the default database user `iris` with password `siri` to extend the database
with a replica on the second server (running on port `9001`):
```
curl --location --request POST 'http://localhost:9021/new-replica' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic c2E6c2lyaQ==' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "dbname": "tutorialdb",
    "username": "iris",
    "password": "siri",
    "host": "localhost",
    "port": 9000,
    "pool": 0
}'
```

We start by downloading the Python script (and this tutorial):
```
cd ~
git clone https://github.com/transceptor-technology/grafana-siridb-http-example.git
cd ./grafana-siridb-http-example
```

Ok, everything is ready to collect data (we configure the other two SiriDB servers later in this tutorial). Before starting the Python script to collect data we must install its dependencies:
```
pip install siridb-connector psutil
```

Start the script. The script accepts arguments which can be viewed with `python mon2siridb.py -h`. If you are following this tutorial then the defaults should be fine.
```
python mon2siridb.py &> mon.log &
```

Let's setup Grafana so we can view what we are collecting. First download and install Grafana:

```
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_6.7.2_amd64.deb
sudo dpkg -i grafana_6.7.2_amd64.deb
```

And install the Grafana-SiriDB-Datasource plugin:
```
cd /var/lib/grafana/plugins/
sudo git clone https://github.com/SiriDB/grafana-siridb-http-datasource.git /var/lib/grafana/plugins/grafana-siridb-http-datasource
```

Start (or restart) Grafana:
```
sudo systemctl restart grafana-server.service
```

Open a browser and go to http://localhost:3000. You should see the following page:

![Grafana login](/png/grafana-login.png?raw=true)

Sign-in by using username `admin` and password `admin`.

Click on ***Add data source*** to create the SiriDB data source. Fill in the form like below (use `siri` as password):

![Grafana add data source](/png/grafana-add-datasource.png?raw=true)

Click on ***Save and test*** should return message that everything is working!

From the menu, click on ***Dashboards*** -> ***Import***

![Grafana menu dashboard import](/png/grafana-menu-dashboard-import.png?raw=true)

Click on ***Upload .json File*** and select the `tutorial-dashboard.json` from this folder.
In the next window you should choose the SiriDB HTTP data source.

![Grafana import dashboard](/png/grafana-import-dashboard.png?raw=true)

After clicking on ***Import*** you should see a dashboard similar to this:

![Grafana tutorial dashboard](/png/grafana-tutorial-dashboard.png?raw=true&v=1)

We can now continue by expanding the database with another pool and use the third server on port `9002`.

```
curl --location --request POST 'http://localhost:9022/new-pool' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic c2E6c2lyaQ==' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "dbname": "tutorialdb",
    "username": "iris",
    "password": "siri",
    "host": "localhost",
    "port": 9000
}'
```

In the dashboard you should see the new server. The status for the existing servers includes ***re-indexing*** while
the series are spread across the pools.

![Grafana re-indexing](/png/grafana-re-indexing.png?raw=true)

Wait until the status for all three server is ***running*** and then create another replica on the fourth server (on port `9003`):
```
curl --location --request POST 'http://localhost:9023/new-replica' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic c2E6c2lyaQ==' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "dbname": "tutorialdb",
    "username": "iris",
    "password": "siri",
    "host": "localhost",
    "port": 9000,
    "pool": 1
}'
```

The dashboard should show the fourth server with status ***synchronizing***

![Grafana synchronizing](/png/grafana-synchronizing.png?raw=true)

From this point it should be rather easy to create your own Grafana Dashboard by using a SiriDB database.
As an example we will add two extra graphs for Disk IO counters.

If you want to select series based on regular expression, then a best practice is to create a dynamic group.
For the current example we create the following two groups:

**`disk_io_counters_read_bytes`**
```
curl --location --request POST 'http://localhost:9020/query/tutorialdb' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic aXJpczpzaXJp' \
--header 'Content-Type: text/plain' \
--data-raw '{
	"q": "create group `disk_io_counters_read_bytes` for /.*disk_io_counters_read_bytes/",
	"t": "ms"
}'
```
**`disk_io_counters_write_bytes`**
```
curl --location --request POST 'http://localhost:9020/query/tutorialdb' \
--header 'Content-Type: application/json' \
--header 'Authorization: Basic aXJpczpzaXJp' \
--header 'Content-Type: text/plain' \
--data-raw '{
	"q": "create group `disk_io_counters_write_bytes` for /.*disk_io_counters_write_bytes/",
	"t": "ms"
}'
```

Go back to Grafana and click on ***Add panel*** -> **Add Query**.

![Grafana graph](/png/grafana-add-panel.png?raw=true&v=1)

At ***select*** fill in ``` `disk_io_counters_read_bytes` ```, choose ***max*** as aggregation and enable ***Diffps***.

![Grafana read bytes](/png/grafana-add-read-bytes.png?raw=true&v=2)

On the General tab you can change the panel title to "Disk IO counters (read bytes)".

![Grafana graph](/png/grafana-add-panel-add-title.png?raw=true&v=1)

Repeat these steps for the ***write*** counters and when you are finished you should have the following result:

![Grafana disk io counters bytes](/png/grafana-disk-io-counters-bytes.png?raw=true)

I hope this tutorial was helpful and I am looking forward to hear what you can create by using Grafana and SiriDB!
