# Live Bitcoin Implied Volatility Charts

![image](https://user-images.githubusercontent.com/313480/140698710-91c1e568-7825-46ad-9656-086c3dae32c5.png)

Implied volatility (IV) is basically a metric used to forecast what the market thinks about the future price movements of an option's underlying stock. IV is a dynamic figure that changes based on activity in the options marketplace. Usually, when implied volatility increases, the price of options will increase as well, assuming all other things remain constant. When implied volatility increases after a trade has been placed, it’s good for the option owner and bad for the option seller. IV is useful because it offers traders a general range of prices that a security is anticipated to swing between and helps indicate good entry and exit points. IV isn’t based on historical pricing data on the stock. Instead, it’s what the marketplace is “implying” the volatility of the stock will be in the future, based on price changes in an option.


Using LedgerX public websocket API we're able to demonstrate using Redis as a multipurpose datastore.
Using a server side Lua script to check the received updates counter we can appropriately publish PUBSUB messages to the
listening [Bokeh](https://docs.bokeh.org/en/latest/) app and store the bid/ask prices to a
[Redis Time Series](https://oss.redislabs.com/redistimeseries/) data type atomically.

The Bokeh app displays the implied volatility calculated from the best bid and offer prices received over websocket.
We're using the Black-Scholes formula implemented by the [vollib](http://vollib.org/) library.

![image](https://user-images.githubusercontent.com/313480/140698675-08c8728f-a92b-426f-9552-aeb9bdfa22fd.png)


We get the price of bitcoin from polling the coinbase API every 3 seconds.

This allows traders to do further analysis and find opportunities in possible mispricings in the volatility component of
the options pricing model.

![architectural diagram](https://i.imgur.com/Yy6aSqQ.png)

When you run `ledgerx_ws.py` a websocket connection is opened with LedgerX and sends data from the incoming messages to a Lua script on the Redis instance that adds new data to a [Time Series structure](https://oss.redis.com/redistimeseries/) and atomically publishes the data to subscribers:

```
redis.call('SETNX', KEYS[1], 0)
local prev_clock = redis.call('GETSET', KEYS[1], ARGV[1])
local clock = tonumber(ARGV[1])
prev_clock = tonumber(prev_clock)
if(clock > prev_clock) then
    redis.call('PUBLISH', ARGV[2], ARGV[5])
    redis.call('TS.ADD', KEYS[2], '*', ARGV[3], 'DUPLICATE_POLICY LAST')
    redis.call('TS.ADD', KEYS[3], '*', ARGV[4], 'DUPLICATE_POLICY LAST')
    return 1
else
    return 0
end
```
The subscribers listening to the Redis PUBSUB channel are charts generated by a [Bokeh server app](https://docs.bokeh.org/en/latest/docs/user_guide/server.html).

### Data Access
In this app we record bid ask prices to Time Series structures with keys formatted like so: `<contract_id>:[bid|ask]` ie., `22227599:ask`

you can access the time series data with the `TS.RANGE` command from the redis-cli ([docs](https://oss.redis.com/redistimeseries/commands/)):
```
127.0.0.1:6379> TS.RANGE "22227599:ask" - +
1) 1) (integer) 1639378065269
   2) 65500
2) 1) (integer) 1639378209405
   2) 65500
3) 1) (integer) 1639378219445
   2) 65500
```
To subscribe to the live data being stored to the time series structs, simply subscribe to this wildcard channel: `*.1`, like so:

```
127.0.0.1:6379> PSUBSCRIBE *.1
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "*.1"
3) (integer) 1
1) "pmessage"
2) "*.1"
3) "22229255.1"
4) "{\"bid\": 14600, \"ask\": 55900, \"contract_type\": 1, \"clock\": 46971, \"bid_size\": 2, \"type\": \"book_top\", \"contract_id\": 22229255, \"ask_size\": 1}"
1) "pmessage"
2) "*.1"
3) "22230741.1"
4) "{\"bid\": 300000, \"ask\": 387100, \"contract_type\": 1, \"clock\": 316717, \"bid_size\": 10, \"type\": \"book_top\", \"contract_id\": 22230741, \"ask_size\": 5}"
```
The subscriber will recieve JSON encoded messages with the contract id, best bid, ask and top of the book order sizes.


## Getting Started

### Step 1. Run Redis container

```
 docker run -d -p 6379:6379 redislabs/redismod
```

Thhis will run both Redis and Redis modules like [Redis Time Series](https://oss.redislabs.com/redistimeseries/)

### Step 2. Clone the repository

```
 git clone https://github.com/redis-developer/bitcon-implied-volatility
```

### Step 3. Install the dependencies

Install the below dependencies in requirements.txt in venv

```
 pip3 install -r requirements.txt`

```

### Step 4. Create a LedgeX API key

Create a LedgerX api key (https://app.ledgerx.com/profile/api-keys) and copy to a file in root of project named "secret"

### Step 5.  Execute the script

Run `ledgerx_ws.py` which is the script consuming the websocket stream from LedgerX

```
 python3 ledgerx_ws.py
```

### Step 5. Run Bokeh server application

Run the below command from the project root to start up the Bokeh server application and open a web browser to
the local URL.

```bash
  bokeh serve --show iv_app.py
```

![image](https://user-images.githubusercontent.com/313480/140698568-d92a7020-db73-47b9-ad2a-fc896d52c897.png)


There are pre-cache files `contracts.pkl` and `id_table.json` which are loaded so no authenticated requests are needed.
If you have a LedgerX account and API key, you can create a file named `secret` with the API key on the first line which
will enable authenticated API queries.

## Using Docker

### Pre-requisite:
- Install docker: https://docs.docker.com/engine/install/

- Run the command to avoid background save failure under low memory condition

```bash
  sysctl vm.overcommit_memory=1
```

- Install docker-compose: https://docs.docker.com/compose/install/

If you don't configure docker to be used without sudo you'll have to add sudo in front of any docker command

- To build image:

```
 sudo docker build -t iv_app:dev .`
```

- To run image interactively, mapping port 5006 from the container to 5006 on your local machine:

`sudo docker run -it -p 5006:5006 -w "/implied-vol-plot" -v "$(pwd):/implied-vol-plot" iv_app:dev bash`

- To run it in the background in detached mode:

`sudo docker run -d -p 5006:5006 -w "/implied-vol-plot" -v "$(pwd):/implied-vol-plot" iv_app:dev`

### Using Docker-compose

- To start services:
`docker-compose up` #can add -d flag to run in background

- To stop services:

`docker-compose down`

To run a specific service interactively:

`docker-compose exec <name-of-service-in-docker-compose-yaml> sh`
