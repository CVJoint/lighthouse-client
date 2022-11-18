# Lighthouse Client - Docker

The official Lighthouse client [sigp/lighthouse](https://hub.docker.com/r/sigp/lighthouse) includes the relevant configuration for the `gnosis` network, and can be used with the flag `--network gnosis`.

Images are referenced under the following pattern `sigp/lighthouse:{image-tag}` with the `image-tag` referring to the image available on Docker Hub.

Most users should use the `latest-modern` tag, which corresponds to the latest stable release of Lighthouse with optimizations enabled. If you are running on older hardware then the default latest image bundles a portable version of Lighthouse which is slower but with better hardware compatibility.

## Lighthouse reference

- General: https://lighthouse-book.sigmaprime.io/

# Starting Lighthouse on mainnet

1. Create your folder structure:

```
mkdir -p /home/$USER/gnosis/el-client
mkdir -p /home/$USER/gnosis/cl-client/data
mkdir /home/$USER/gnosis/cl-client/keystores
mkdir /home/$USER/gnosis/cl-client/validators
mkdir /home/$USER/gnosis/jwtsecret
```

2. Create your docker-compose file with your favorite text editor in `/home/$USER/gnosis/docker-compose.yml`:

```
version: "3"
services:

  el-client:
    hostname: el-client
    container_name: el-client
    image: nethermind/nethermind:latest
    restart: always
    stop_grace_period: 1m
    command: |
      --config xdai
      --datadir /data
      --JsonRpc.Enabled true
      --JsonRpc.Host 192.168.32.100
      --JsonRpc.Port 8545
      --JsonRpc.JwtSecretFile /jwtsecret.json
      --JsonRpc.EngineHost 192.168.32.100
      --JsonRpc.EnginePort 8551
      --Merge.Enabled true
    networks:
      gnosis_net:
        ipv4_address: 192.168.32.100
    ports:
      - "30303:30303/tcp"
      - "30303:30303/udp"
    volumes:
      - /home/$USER/gnosis/el-client:/data
      - /home/$USER/gnosis/jwtsecret/jwtsecret.json:/jwtsecret.json
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    logging:
      driver: "local"

  cl-client:
    hostname: cl-client
    container_name: cl-client
    image: sigp/lighthouse:latest-modern
    restart: always
    depends_on:
      - el-client
    command: |
      lighthouse beacon_node
      --network gnosis
      --discovery-port 12000
      --port 13000
      --execution-endpoint http://192.168.32.100:8551
      --execution-jwt /jwtsecret.json
      --datadir /data
      --http
      --http-address 192.168.32.101
      --enr-address $WAN_IP
      --enr-udp-port 12000
      --target-peers 80
      --debug-level info
      --suggested-fee-recipient $FEE_RECIPIENT
      --checkpoint-sync-url $CHECKPOINT_URL
    networks:
      gnosis_net:
        ipv4_address: 192.168.32.101
    ports:
      - "12000:12000/udp" #p2p
      - "13000:13000/tcp" #p2p
    volumes:
      - /home/$USER/gnosis/cl-client/data:/data
      - /home/$USER/gnosis/jwtsecret/jwtsecret.json:/jwtsecret.json
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    logging:
      driver: "local"

  validator:
    hostname: validator
    container_name: validator
    image: sigp/lighthouse:latest-modern
    restart: always
    depends_on:
      - cl-client
    command: |
      lighthouse validator_client
      --network gnosis
      --enable-doppelganger-protection
      --validators-dir /data/validators
      --beacon-nodes http://192.168.32.101:5052
      --suggested-fee-recipient $FEE_RECIPIENT
      --graffiti $GRAFFITI
    networks:
      gnosis_net:
        ipv4_address: 192.168.32.102
    volumes:
      - /home/$USER/gnosis/cl-client/validators:/data/validators
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    logging:
      driver: "local"

networks:
  gnosis_net:
    ipam:
      driver: default
      config:
        - subnet: 192.168.32.0/24
```

3. Add an `.env` file with your WAN IP (`curl https://ipinfo.io/ip`), fee recepient (your gnosis address), graffiti, and checkpoint url `/home/$USER/gnosis/.env`.

```
WAN_IP:123.456.789.012
FEE_RECIPIENT=0x0000000000000000000000000000000000000000
GRAFFITI=gnosischain/lighthouse
CHECKPOINT_URL=https://rpc-gbc.gnosischain.com/
```

4. Add your keystores in `/home/$USER/gnosis/cl-client/keystores` and their password in a file `/home/$USER/gnosis/cl-client/keystores/password.txt` to get this file structure:

Note, keystores MUST follow one of these file names

- `keystore-m_12381_3600_[0-9]+_0_0-[0-9]+.json` The format exported by the `eth2.0-deposit-cli` library ([source](https://github.com/sigp/lighthouse/blob/2983235650811437b44199f9c94e517e948a1e9b/common/account_utils/src/validator_definitions.rs#L402))
- `keystore-[0-9]+.json` The format exported by Prysm ([source](https://github.com/sigp/lighthouse/blob/2983235650811437b44199f9c94e517e948a1e9b/common/account_utils/src/validator_definitions.rs#L411))

```
/home/$USER/gnosis/
├── docker-compose.yml
├── .env
├── jwtsecret/
├── el-client/
└── cl-client/
    ├── data/
    ├── keystores/
    │   ├── keystore-001.json
    │   ├── keystore-002.json
    │   └── password.txt
    └── validators/
```

5. Create a new `./jwtsecret` token:

```
openssl rand -hex 32 | tr -d "\n" > /home/$USER/gnosis/jwtsecret/jwtsecret.json
```

6. Import your validators:

```
docker run --rm --volume /home/$USER/gnosis/cl-client/keystores:/keystores --volume /home/$USER/gnosis/cl-client:/data sigp/lighthouse:latest-modern lighthouse account validator import --network gnosis --password-file /keystores/password.txt --reuse-password --directory /keystores --datadir /data
```

7. Start `docker-compose.yml` services from this repository

```
cd /home/$USER/gnosis
docker-compose up -d
```

8. Check your logs for each service (`el-client`, `cl-node`, or `cl-validator`) with:

```
docker logs -f --tail 500 <service>
```

9. To update, just pull the new images, then stop and restart your docker-compose file:
```
cd /home/$USER/gnosis
docker-compose pull
docker-compose stop
docker-compose up -d
```
