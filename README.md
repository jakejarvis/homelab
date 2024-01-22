# homelab

Docker Compose service definitions for the various apps I have running on my home network.

## Usage

After setting up all of the necessary environment variables (remember to `cp .env.example .env` first), everything (besides some of the weirdness discussed below) _should_ just kinda work, I think...?

```sh
docker compose up -d
```

## Services

> 🌎 = Inbound access is handled by a [Traefik](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) container, proxied via a DigitalOcean VPS running [Caddy](https://caddyserver.com/) for access from anywhere. This negates the need to open any ports on the router. (More on this soon.)
>
> 🔐 = Protected by [Authelia](https://www.authelia.com/) single sign-on.
>
> 🔓 = Publically accessible but [OAuth](https://openid.net/developers/how-connect-works/) (also [via Authelia](https://www.authelia.com/configuration/identity-providers/open-id-connect/)) is used for admin logins.
>
> 🚇 = Outgoing WAN traffic is tunneled through a [WireGuard client container](https://github.com/linuxserver/docker-wireguard) to [Mullvad VPN](https://mullvad.net/en/).

- 🌎 🔓 [Gitea](https://github.com/go-gitea/gitea)
- 🌎 🔓 [MinIO](https://github.com/minio/minio)
- 🌎 🔐 🚇 [Sonarr](https://github.com/linuxserver/docker-sonarr)
- 🌎 🔐 🚇 [Radarr](https://github.com/linuxserver/docker-radarr)
- 🌎 🔐 🚇 [Prowlarr](https://github.com/linuxserver/docker-prowlarr)
- 🌎 🔐 🚇 [Bazarr](https://github.com/linuxserver/docker-bazarr)
- 🌎 🔐 🚇 [qBittorrent](https://github.com/linuxserver/docker-qbittorrent)
- 🌎 🔐 [Tautulli](https://github.com/Tautulli/Tautulli)
- 🌎 🔐 [Homepage](https://github.com/gethomepage/homepage)
- 🌎 🔐 [Portainer](https://docs.portainer.io/start/install-ce/server/docker/linux)
- 🌎 🔐 [Munin](https://github.com/aheimsbakk/container-munin)
- 🌎 🔐 [Dozzle](https://github.com/amir20/dozzle)
- 🚇 [Flaresolverr](https://github.com/FlareSolverr/FlareSolverr)

## Notes

1. [Docker secrets](https://docs.docker.com/engine/swarm/secrets/#use-secrets-in-compose) are used to store/read the WireGuard private key ~~and Plex Pass claim token~~. These are stored as files in the `secrets` subdirectory here and mounted to `/run/secrets/secret_name` in the containers.
  - Getting the keys for Mullvad is pretty awkward — you need to generate configuration files (with a new private key if needed) from the [Mullvad dashboard](https://mullvad.net/en/account/#/wireguard-config), extract the ZIP archive it spits out, and then open any one of the .conf files **in a text editor** to extract the `PrivateKey`. Read more: https://github.com/qdm12/gluetun/wiki/Mullvad#wireguard-only
  - In the same .conf file(s), the hard-coded `PrivateKey` can now be removed. Replace it with the following `PostUp` command so that WireGuard knows to refer to the Docker secret for it instead:

```diff
[Interface]
- PrivateKey = xxxxxx
+ PostUp = wg set %i private-key <(cat /run/secrets/wg_private_key)
```

2. [Port forwarding](https://mullvad.net/en/account/#/port-forwarding) is probably the biggest benefit of using Mullvad and can speed up BT downloads bigly. Mullvad can assign a random one (between 40000ish and 60000ish) pointed towards the WireGuard keypair used above. Setting `VPN_FORWARDED_PORT` to this number will tell both WireGuard and qBittorrent to open the port and use it for P2P connections.

3. Filesystem permissions get super tricky and frustrating very quickly. Carefully override `UID` and `GID` if necessary (both default to `1000` for most containers, which usually belong to the first non-root account created on the host) and, um... good luck.

4. The conflicts between Docker and UFW are [_far_ more severe](https://github.com/chaifeng/ufw-docker) than I realized. Thankfully, nothing here is important enough to cause me to lose sleep at night, so poking these holes in the wall seemed to help solve/cover up most issues...

```sh
ufw route allow from 172.17.0.0/12 to 172.17.0.0/12
```

## Host preparation

These notes are for my own reference and are probably band-aids for edge cases that may or may not apply to anyone else. In reality, they'll probably cause you more problems and introduce security holes if you're hosting anything other than _~~completely legally obtained TV shows~~_ Linux ISOs...

Everything here assumes the host is running something Debian-based (because [it is](https://www.raspberrypi.com/software/)).

#### Install upstream Docker

https://docs.docker.com/engine/install/debian/

```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install the Docker packages
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Login to GitHub Container Registry

[Create a PAT (classic)](https://github.com/settings/tokens) with the `read:packages` scope.

```sh
export CR_PAT=TOKEN_FROM_THERE
echo $CR_PAT | docker login ghcr.io -u jakejarvis --password-stdin
```

## License

[MIT](../LICENSE)
