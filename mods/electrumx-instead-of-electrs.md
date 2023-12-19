# Use ElectrumX Electrum server instead of Electrs

[Electrs](https://github.com/romanz/electrs) is the default Electrum server implementation for Umbrel. Its installation is required by many Bitcoin apps such as BTCPay Server or mempool explorer. While this implementation works fine for basic queries, it is known to have troubles serving heavier and more frequent queries, such as loading wallets with big history or browsing a blockchain explorer.

Electrs can be replaced with another Electrum server implementation while keeping the dependent apps working. In this case, the replacement is [ElectrumX](https://github.com/spesmilo/electrumx.git).

## What to consider
- ElectrumX serves queries which knock out Electrs
- At block height 821900, at the end of 2023, ElectrumX index takes 95 GB of space
- The index takes few days to build no matter what ETA logs say
- ElectrumX doesn't serve queries until the index is built
- Once the index is built, it takes up to 10 minutes for ElectrumX to start serving queries.
Disabling persistent mempool for Bitcoin node speeds up the startup time after system reboot. 

## Doing the replacement

To start, you must have Electrs installed. It doesn't matter if it is synchronized or not.

First, stop the Electrs app:

```bash
sudo ./umbrel/scripts/app stop electrs
```

Once Electrs is stopped, edit its `docker-compose` file with the editor of your choice. The file path is `./umbrel/app-data/electrs/docker-compose.yml`.

In the `docker-compose.yaml`, find the `electrs:` section and comment it by adding `#` to each line, to make it look like this:

```
  # electrs:
  #   image: getumbrel/electrs:v0.10.1@sha256:79cdac6a7bcdb51f8e4c6450411e42a90db3665939b6e33cbfb1c30ef3df00d3
  #   restart: always
  #   environment:
  #     ELECTRS_LOG_FILTERS: "INFO"
  #     ELECTRS_NETWORK: "${APP_BITCOIN_NETWORK_ELECTRS}"
  #     ELECTRS_DAEMON_RPC_ADDR: "${APP_BITCOIN_NODE_IP}:${APP_BITCOIN_RPC_PORT}"
  #     ELECTRS_DAEMON_P2P_ADDR: "${APP_BITCOIN_NODE_IP}:${APP_BITCOIN_P2P_PORT}"
  #     ELECTRS_ELECTRUM_RPC_ADDR: "0.0.0.0:${APP_ELECTRS_NODE_PORT}"
  #     ELECTRS_SERVER_BANNER: "Umbrel Electrs (${APP_VERSION})"
  #     ELECTRS_DB_DIR: "/data/db"
  #   volumes:
  #     - "${APP_BITCOIN_DATA_DIR}:/data/.bitcoin:ro"
  #     - "${APP_DATA_DIR}/data/electrs:/data"
  #   ports:
  #     - "${APP_ELECTRS_NODE_PORT}:${APP_ELECTRS_NODE_PORT}"
  #   networks:
  #     default:
  #       ipv4_address: $APP_ELECTRS_NODE_IP
```

Then, add the replacement section with ElectrumX config above or below the commented section. Make sure to keep the space padding.

```
  electrumx:
    image: lukechilds/electrumx
    restart: always
    stop_grace_period: 300s
    environment:
      DAEMON_URL: "http://${APP_BITCOIN_RPC_USER}:${APP_BITCOIN_RPC_PASS}@${APP_BITCOIN_NODE_IP}:${APP_BITCOIN_RPC_PORT}"
      COIN: "BitcoinSegwit"
      LOG_LEVEL: "debug"
      PEER_DISCOVERY: "off"
      SERVICES: "tcp://:50001"
    volumes:
      - "${APP_DATA_DIR}/data/electrumx:/data
    ports:
      - "${APP_ELECTRS_NODE_PORT}:50001"
    networks:
      default:
        ipv4_address: $APP_ELECTRS_NODE_IP
```

Here we use [Luke Childs' ElectrumX Docker image](https://github.com/lukechilds/docker-electrumx) which will run the server on the same address and port as Electrs, using `./umbrel/app-data/electrs/data/electrumx` directory to store the index.

Save the file and start the modified app:


```bash
sudo ./umbrel/scripts/app start electrs
```

Check that ElectrumX has started and indexing by checking the logs:

```bash
sudo ./umbrel/scripts/app compose electrs logs -f electrumx
```

Hit `Ctrl+C` multiple times to exit the log printer.

## After the replacement

The Electrs web app will not show ElectrumX sync status, so check the logs periodically and wait for the server to build the index. Once the index is built, it takes up to 10 minutes for ElectrumX to start serving queries.

The server is accessible on the same IP, port and Tor hostname as Electrs has been, no dependent apps or wallets reconfiguration is required.

Browse a blockchain explorer, connect your wallet, try arbitrary address or transaction lookups to evaluate ElectrumX performance. Use it for a week or two. If you find it working better than Electrs for you, remove the Electrs index (`./umbrel/app-data/electrs/data/electrs`) to free some space.

If you decided not to run ElectrumX and return to Electrs, do the same as you did for the replacement, but uncomment the `electrs` section and remove `electrumx` one.
