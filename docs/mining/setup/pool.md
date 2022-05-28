> Community support available in [#host-a-pool](https://discord.gg/kxbrHVwnm5) on Discord.

# Hosting a Pool on Ergo


## ErgoStratum

1. Setup a Node
2. Setup [ErgoStratumServer](https://github.com/mhssamadani/ErgoStratumServer) (the pool server)
3. Point [ergo-nomp](https://github.com/btclinux/ergo-nomp), a Node open mining portal configufed for ergo to both. 

For the  Miner Connection + Work Management, [ErgoStratumProxy](https://github.com/mhssamadani/ErgoStratumProxy) is also available.


## Miningcore

The following tutorial is written for Linux, see [this page if you're using Windows 10](pool_win.md)

### Download [miningcore](https://github.com/oliverw/miningcore)  

- You must deploy a working [**PostgreSQL** database](https://www.postgresql.org/download/)

### Install and configure PostgreSQL

If you are deploying in production, you REALLY need to be careful on I/O and disk/cpu/memory as a public Miningcore API will pound on your database.  Keep all things default for now, here's a [good simple article](https://www.postgresqltutorial.com/install-postgresql/)

Installing Miningcore will depend if you are on windows or Linux. Pay close attention to the dependencies needed as depicted in the https://github.com/oliverw/miningcore README.md.  Stay away from docker, you don't need it if you are just kicking the tires and want to mine to your own pool.

### Load the schema  

Miningcore operates under the schema `miningcore` by default.  

**Login to postgres** 

```bash
sudo -u postgres psql
```

**Create schema**

```SQL
CREATE ROLE miningcore WITH LOGIN ENCRYPTED PASSWORD 'your-secure-password';
CREATE DATABASE miningcore OWNER miningcore;
```

**Load the schema.** 

> You MUST be able to connect to your database with the `psql` command before trying this.  

As the postgres operating system user, run:

```SQL
psql -d miningcore -f miningcore/src/Miningcore/Persistence/Postgres/Scripts/createdb.sql
```

**Apply partitions for the shares table. **

Do this up front even if you dont think you will need it:

```SQL
psql -d miningcore -f miningcore/src/Miningcore/Persistence/Postgres/Scripts/createdb_postgresql_11_appendix.sql
```

**Create your new pool. **

You do this ONCE for each new pool you create. 

```SQL
CREATE TABLE shares_mypool1 PARTITION OF shares FOR VALUES IN ('mypool1');
```

`mypool1` becomes the unique identifier for your pool. Name it wisely 

Configure a pool config named <something>.json, place it in the build directory of miningcore.  [Here's a sample file to use](https://www.getblok.io/wp-content/uploads/2022/03/ergo1.zip)  (extract the GZ and make sure its named <something>.json)



### Start the pool 

You should configure it to auto start via a startup script. 

```
cd build
Miningcore -c <your config>.json
```

We use a different startup configuration that is pool specific so you may have to play around with it.  The JSON config specifies the log files you should look at for startup errors/issues/etc.  If you got this far, below is the startup log message you should get indicating you are running a healthy miningcore ergo pool:

**Indicates your Node is online and synced**

```
[2022-03-16 14:26:12.9080] [I] [ergo1] All daemons online
[2022-03-16 14:26:12.9345] [I] [ergo1] Daemon is synced with blockchain
```

**Indicates Pool is online**
```
[2022-03-16 14:26:14.4346] [I] [ergo1] Pool Online
```
Indicates the diff setting, your fee, and that its a go:
```
Mining Pool:            <YOUR POOL NAME>
Coin Type:              ERG [ERG]
Network Connected:      <testnet|mainnet>
Detected Reward Type:   POW
Current Block Height:   <BLOCKHEIGHT>
Current Connect Peers:  5
Network Difficulty:     <NETWORK DIFF>
Network Hash Rate:      <NETWORK HASHRATE>
Stratum Port(s):        3056, 4056, 3156, 4156  <THIS SHOULD BE THE PORTS YOU HAVE CONFIGURED>
Pool Fee:               <YOUR FEE>
```

Pay close attention to the stats above in regards to the network. If your diff setting is wrong, the network difficulty wont be right.

> If you host your miner, pool, or node on separate machines, you'll need to open ports to allow that traffic.

**Illustration of initial mining traffic:**

- Miner -> 
    - (connects on your defined Stratum port i.e. `3746`) -> 
- pool server -> 
    - connects to node RPC (`9053`, mainnet or `9052`, testnet) -> 
- node
  
Once connections are established, traffic becomes bi-directional.

**You will not need to open ports if any of the above layers are on the same machine.**  Internal traffic runs via the internal loop (localhost). 

Typically, machines block incoming traffic (but allow outgoing) by default. If you're just testing on LAN, you'll simply need to open these ports on your OS's firewall.  But when you get to WAN, you'll additionally need to set up port forwarding on your network router.