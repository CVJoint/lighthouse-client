# Lighthouse Client - Docker

This project uses the official lighthouse client [sigp/lighthouse](https://hub.docker.com/r/sigp/lighthouse) which includes the relevant configuration for the `gnosis` network.

Images are referenced under the following pattern `sigp/lighthouse:{image-tag}` with the `image-tag` referring to the image available on Docker Hub.

Most users should use the `latest-modern` tag, which corresponds to the latest stable release of Lighthouse with optimizations enabled. If you are running on older hardware then the default latest image bundles a portable version of Lighthouse which is slower but with better hardware compatibility.

Pulling the latest lighthouse beacon client would look like this: 

```
docker pull sigp/lighthouse:latest-modern
```

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

2. Add an `.env` file with your WAN IP (`curl https://ipinfo.io/ip`), fee recepient (your gnosis address), and graffiti `/home/$USER/gnosis/.env`.

```
WAN_IP:123.456.789.012
FEE_RECIPIENT=0x0000000000000000000000000000000000000000
GRAFFITI=gnosischain/lighthouse
```

3. Add your keystores in `/home/$USER/gnosis/cl-client/keystores` and their password in a file `/home/$USER/gnosis/cl-client/keystores/password.txt` to get this file structure:

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

4. Create a new `./jwtsecret` token:

```
openssl rand -hex 32 | tr -d "\n" > /home/$USER/gnosis/jwtsecret/jwtsecret
```

5. Import your validators:

```
docker run --rm --volume /home/$USER/gnosis/cl-client/keystores:/keystores --volume /home/$USER/gnosis/cl-client:/data sigp/lighthouse:latest-modern lighthouse account validator import --network gnosis --password-file /keystores/password.txt --reuse-password --directory /keystores --datadir /data
```

4. Start `docker-compose.yml` services from this repository

```
cd /home/$USER/gnosis
docker-compose up -d
```

5. Check your logs for each service (`el-client`, `cl-node`, or `cl-validator`) with:

```
docker logs -f --tail 500 <service>
```
