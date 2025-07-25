# Brook Protocol

**joker** is just a supervisor service to run the brook in background

## Brook

### Server

```bash
joker brook server --listen <SERVER_LISTEN_IP_ADDR>:<SERVER_LISTEN_PORT> -p '<STRONG_PASSWORD>'
```

### Client

```bash
joker brook client -s <DEST_SERVER_IP>:<DEST_SERVER_PORT> -p '<STRONG_PASSWORD>' --socks5 <LOCAL_LISTEN_ADDR>:<LOCAL_LISTEN_PORT>
```

## Brook + WebSocketSecure

### WSS Server

```bash
joker brook wssserver --domainaddress <FQDN>:<PORT> --password '<STRONG_PASSWORD>' --cert /path/to/<FQDN>/fullchain.pem --certkey /path/to/<FQDN>/privkey.pem
```

### WSS Client

```bash
joker brook --log /tmp/brook.log --tag wssserver:germany wssclient --wssserver wss://<FQDN>:<PORT> --password '<STRONG_PASSWORD>' --tlsfingerprint chrome --socks5 <LOCAL_LISTEN_ADDR>:<LOCAL_LISTEN_PORT>
```

### Client systemd service

```conf
# /etc/systemd/system/de-brook-socks5-1080.service
[Unit]
Description=Brook Client Service
After=network.target nss-lookup.target network-online.target

[Service]
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/root/.nami/bin/brook --log /tmp/de-brook-socks5.log --tag wssserver:germany wssclient --wssserver wss://<FQDN>:<PORT> --password '<STRONG_PASSWORD>' --socks5 <LOCAL_LISTEN_ADDR>:<LOCAL_LISTEN_PORT>
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target

```

This below systemd service is going to call the brook wss server through another socks5 proxy server (Chain Proxying)

```conf
# /etc/systemd/system/nl-brook-socks5-20808.service
[Unit]
Description=Brook Client Service
After=network.target nss-lookup.target network-online.target

[Service]
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/root/.nami/bin/brook --dialWithSocks5 127.0.0.1:10808 --log /tmp/nl-brook-socks5.log --tag wssserver:netherland wssclient --wssserver wss://<FQDN>:<PORT> --password '<STRONG_PASSWORD>' --socks5 <LOCAL_LISTEN_ADDR>:<LOCAL_LISTEN_PORT>
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
```

## Relay Servers

### Relay

```mermaid
graph LR;
    A[Client] -->|brook| B[Relay Server] --> C[Destination Server];

```

```conf
Description=Brook Relay To <Foreign Server>
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/root/.nami/bin/brook relay --from <RELAY_SERVER_LISTEN_ADDR>:<RELAY_SERVER_LISTEN_PORT> --to <SERVER_ADDR>:<SERVER_PORT>
[Install]
WantedBy=multi-user.target
```

### Relay OverBrook

This mean Using a Relay server which is using a brook to relay the client, (brook through a brook from relay server)

```mermaid
graph LR;
    A[Client] -->|brook| B[Relay Server] -->|brook| C[Destination Server];

```

```conf
[Unit]
Description=Brook OverRelayBrook To Hetzner
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/root/.nami/bin/brook relayoverbrook --from <RELAY_SERVER_LISTEN_ADDR>:<RELAY_SERVER_LISTEN_PORT> -p '<STRONg_PASSWD>' --server <SERVER_ADDR>:<SERVER_PORT> --to <SERVER_ADDR>:<SERVER_PORT>
[Install]
WantedBy=multi-user.target

```

### Relay Clients

### Brook Relay

- Iran -> Free Internet Server

    ```bash
    joker brook client -s  <RELAY_SERVER_LISTEN_ADDR>:<RELAY_SERVER_LISTEN_PORT> -p '<STRONg_PASSWD>' --socks5 <LOCAL_LISTEN_ADDR>:<LOCAL_LISTEN_PORT>
    ```

### Brook RelayOverBrook

- Iran -> Free Internet Server

    ```bash
    joker brook client -s  <RELAY_SERVER_LISTEN_ADDR>:<RELAY_SERVER_LISTEN_PORT> -p '<STRONg_PASSWD>' --socks5 <LOCAL_LISTEN_ADDR>:<LOCAL_LISTEN_PORT>
    ```

Links:

[Brook in Github](https://github.com/txthinking/brook)
