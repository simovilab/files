# files

Copyparty file hosting for SIMOVI projects at **https://files.simovi.org**, using the official [`copyparty/ac`](https://hub.docker.com/r/copyparty/ac) image and an **existing Traefik** [instance](https://github.com/simovilab/traefik/).

## Access model

- Anyone can browse directories and download files without logging in.
- Direct links such as `https://files.simovi.org/logos/simovi.png` are public and work for embedded images.
- Uploading, creating folders, moving, renaming, overwriting, and deleting require an account with the corresponding permission.
- The browser UI is public; **management actions**, not the whole UI, are password-protected. Do not add a global Traefik BasicAuth middleware, which would also block public files.
- There are no default credentials or self-service registrations. Administrators create accounts in the private configuration.

## Server prerequisites

- Docker Engine and Docker Compose v2 or newer.
- Traefik already attached to the external Docker network `traefik_proxy`, with its Docker provider enabled.
- HTTP and HTTPS entrypoints (defaults: `web` and `websecure`) and a configured Let's Encrypt resolver (default name: `letsencrypt`).
- DNS for `files.simovi.org` pointing to this server. Any AAAA record must also point to a working IPv6 endpoint.
- Traefik's ACME challenge prerequisites satisfied, including reachable ports for HTTP/TLS challenges or valid DNS challenge credentials.

This project does not create Traefik, configure its ACME resolver, or store certificates. The labels request a certificate from the resolver you already configured. Copyparty has no published host ports.

## Initial setup

Run these commands in the repository on the deployment server:

```sh path=null start=null
cp .env.example .env
cp config/copyparty.conf.example config/copyparty.conf
mkdir -p data state
chmod 700 state
chmod 600 .env config/copyparty.conf
```

Edit `.env`:

- Match `TRAEFIK_HTTP_ENTRYPOINT`, `TRAEFIK_HTTPS_ENTRYPOINT`, and `TRAEFIK_CERT_RESOLVER` to your Traefik configuration.
- Set `COPYPARTY_UID` and `COPYPARTY_GID` to a dedicated non-root host user/group.
- Optionally set `COPYPARTY_DATA_DIR` to an existing absolute storage path.
- Set `COPYPARTY_IMAGE` to a tested official release tag or immutable digest for reproducible production upgrades. `latest` is a convenience default, not an immutable version.

The configured UID/GID must own or have appropriate access to the data and state directories, and must be able to read `config/copyparty.conf`. For the default UID/GID and data path on Linux:

```sh path=null start=null
sudo chown 1000:1000 data state config/copyparty.conf
```

If using another UID/GID or data path, adjust that command accordingly. Do not make the private configuration or state world-readable. On SELinux hosts, apply appropriate container bind-mount labeling.

### Trusted proxy / real client IP

Find Traefik's address on `traefik_proxy`, using its actual container name:

```sh path=null start=null
docker inspect --format '{{with index .NetworkSettings.Networks "traefik_proxy"}}{{.IPAddress}}{{end}}' TRAEFIK_CONTAINER_NAME
docker network inspect traefik_proxy
```

In `config/copyparty.conf`, replace `xff-src: 127.0.0.1/32` with Traefik's address followed by `/32`. Prefer a stable Traefik address; update this value if its address changes. Alternatively, trust the network's exact CIDR only if every container on that network is trusted.

`rproxy: -1` selects the rightmost forwarded address, appropriate when Traefik is the only proxy. If a CDN or another proxy sits in front of Traefik, configure Traefik's trusted forwarded headers and Copyparty's proxy-hop selection for that topology; do not enable unrestricted forwarded-header trust.

The loopback-only example is deliberately restrictive: real client IP logging and IP-based protections will not work correctly through Traefik until you configure this.

## Create accounts

Generate a password hash interactively **before starting the public service**:

```sh path=null start=null
docker compose run --rm --no-deps copyparty -c /cfg/copyparty.conf --ah-cli
```

Enter a strong password at the prompts; do not put it in command arguments, shell history, or `.env`. Copy the generated hash, including its leading `+`, into the `[accounts]` section of `config/copyparty.conf`. Repeat at the next prompt for each account, then leave the interactive tool with Ctrl-D.

For example, replace these illustrative hash placeholders with your generated hashes:

```ini path=null start=null
[accounts]
  admin: +YOUR_GENERATED_HASH
  editor: +ANOTHER_GENERATED_HASH
```

Uncomment the matching permissions in the existing root volume:

```ini path=null start=null
[/]
  /w
  accs:
    r: *
    rwmda: admin
    rwmd: editor
```

Permissions: `r` browse/download, `w` upload/create, `m` move/rename, `d` delete, `a` Copyparty volume administration (including upload information). `rwmd` is sufficient for normal file management. Copyparty's `a` permission does not let someone create server accounts; those remain configuration-managed.

The checked-in template grants only public reads until you add accounts and uncomment their permission lines. Never grant `w`, `m`, or `d` to `*`.

Argon2 hashing uses a salt persisted under `state/copyparty/`. Generate hashes with the same state directory used by the running service. Losing that salt invalidates the existing hashes.

## Start and manage

```sh path=null start=null
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --follow copyparty
```

Open https://files.simovi.org. Use Copyparty's login control with an account's username and password to manage files.

After account or permission changes, restart the service to reload its read-only mounted configuration:

```sh path=null start=null
docker compose restart copyparty
```

If you replace the config file atomically (for example, using some editors or deployment tools), recreate the container so its single-file bind mount points to the new file:

```sh path=null start=null
docker compose up -d --force-recreate copyparty
```

Files in `data/logos/simovi.png` (or the equivalent under your custom data directory) appear at `/logos/simovi.png`. Files copied directly onto the server are rescanned every five minutes. Everything stored under the data directory should be considered public; keep secrets and backups elsewhere.

## Verify before announcing the service

1. In a private browser window, confirm that directory browsing and a known direct file link work without login.
2. Confirm that `http://files.simovi.org` redirects to HTTPS and HTTPS has a valid certificate.
3. Confirm anonymous uploads and folder creation are denied. Also test rename/move and deletion using disposable test files; public access must never permit mutations.
4. Log in as an editor and verify upload, folder creation, rename/move, and deletion of disposable files. Log out and repeat the anonymous checks.
5. Check the startup permission summary: everyone should have **read only**, with write/move/delete assigned only to named accounts.
6. Check logs for correct client IPs and ensure configuration/state cannot be browsed.

The public data tree is served separately from private configuration and state. The container runs non-root with a read-only root filesystem, dropped capabilities, a temporary scratch directory, and rotated Docker logs.

## Backups and upgrades

Back up the public data directory, `config/copyparty.conf`, `.env`, and **all of `state/`**, including authentication salts and sessions. Stop Copyparty while taking a filesystem copy of its databases, or use a consistent snapshot. Keep backups private and test restores.

Update the pinned image in `.env`, pull, and recreate the service during a maintenance window. Repeat the access checks after every upgrade. Do not assume a healthcheck alone proves the permission model is correct.

Private deployment files, runtime state, and public data are ignored by Git. Never commit passwords, hashes, salts, or session databases.

## References

- [Copyparty Docker documentation](https://github.com/9001/copyparty/blob/hovudstraum/scripts/docker/README.md)
- [Copyparty accounts and volumes](https://github.com/9001/copyparty#accounts-and-volumes)
- [Copyparty real-IP configuration](https://github.com/9001/copyparty#real-ip)
