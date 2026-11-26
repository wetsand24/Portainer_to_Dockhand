# Portainer to Dockhand: move your stacks and keep your data

Every command from the video. Dockhand is installed next to Portainer on the same server, and four apps (Vaultwarden, Paperless-ngx, Uptime Kuma and ntfy) are moved over one at a time without losing a password, document or alert. Then Dockhand's own update checks replace Watchtower, and Portainer is removed.

▶️ Video: https://youtu.be/iEQ6XG0rzn8

Tested 2026-11-24 on Ubuntu Server 24.04 with Docker Engine 29.10.0 (Docker's own apt repo), Portainer CE 2.45.4 LTS and
Dockhand v1.0.59. The demo apps: ntfy v2.38.1, Vaultwarden 1.37.5, Paperless-ngx (+ Postgres 18 + Valkey) and
Uptime Kuma 2.5.12. Replace `<...>` with your own values.

## 0. Before you start: write down what you have

```bash
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Label "com.docker.compose.project"}}'
docker volume ls
```
The `com.docker.compose.project` column is each stack's **project name**. Named volumes are called
`<project>_<volume>` (for example `vaultwarden_data`). If you keep the project name the same, the app keeps its data.

## 1. Install Dockhand next to Portainer

This is the "matching paths" form from the Dockhand manual's Quick start, plus one line so Dockhand can see your stack
folders at the same path as on the host:

```bash
sudo mkdir -p /opt/dockhand
sudo docker run -d \
  --name dockhand \
  --restart unless-stopped \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /opt/dockhand:/opt/dockhand \
  -e DATA_DIR=/opt/dockhand \
  -v /opt/stacks:/opt/stacks \
  fnsys/dockhand:v1.0.59
```
- Change `/opt/stacks` to wherever your compose folders live. Use the same path on both sides of the `:`.
- Open `http://<server-ip>:3000`.

## 2. First launch in the browser

1. You get an "Unsecured exposure" warning. Click **Got it**. It comes back on every page load until login is on.
2. **Go to Settings** > Environments > **Add environment**:
   - Name `homelab`
   - Connection type **Unix socket**
   - Socket path `/var/run/docker.sock`
   - Public IP = your server IP

   Then **Test connection** ("Connected! Docker ... - N containers") > **Add**.
3. Settings > **Authentication** > **Users** > **Add user**: enter a username and password > **Create user**.
4. Settings > Authentication > **General**: switch **OFF** to **ON** ("Authentication enabled"). You are signed out; sign in
   with the new user. Creating a user alone does NOT turn login on.

## 3. Folder stacks (made with `docker compose` in a folder): ntfy, Paperless-ngx

Stacks > **Adopt** > browse to `/opt/stacks` > **Scan this folder** > tick the stack (running stacks are NOT
pre-selected) > **Adopt 1 stack**.
- The Source changes from Untracked to Internal.
- Nothing restarts.
- The compose file and `.env` stay where they are.

Dockhand reads the folder's `.env` (for Paperless: `DATA_PATH=/srv/paperless`) and shows it as "Env file" in the editor.
A redeploy keeps the same bind paths.

**Redeploy / Force recreate:** the Redeploy popover has **Pull images** switched ON by default
(`docker compose up -d --remove-orphans --pull always`). With a `:latest` tag, that pulls a newer version. Switch Pull
images OFF if you only want to recreate on the image you already have.

## 4. Stacks made in Portainer's web editor: Vaultwarden

Portainer keeps them in its data volume, one numbered folder per stack:

```bash
sudo ls /var/lib/docker/volumes/portainer_data/_data/compose/
sudo ls /var/lib/docker/volumes/portainer_data/_data/compose/<id>/      # docker-compose.yml  stack.env
```
`stack.env` holds the environment variables you typed into Portainer (for example DOMAIN, SIGNUPS_ALLOWED).

Copy both files into a folder named after the stack. The folder name becomes the project name, so it must be the SAME
name Portainer used:

```bash
P=/var/lib/docker/volumes/portainer_data/_data/compose/<id>
sudo mkdir -p /opt/stacks/vaultwarden
sudo cp $P/docker-compose.yml /opt/stacks/vaultwarden/compose.yaml
sudo cp $P/stack.env /opt/stacks/vaultwarden/.env
sudo chmod 600 /opt/stacks/vaultwarden/.env
```
1. In Portainer: Stacks > `vaultwarden` > **Stop this stack**. Portainer removes the containers and keeps the volumes.
2. In Dockhand: Stacks > Adopt > Scan this folder. A stopped stack is pre-selected. Click Adopt 1 stack > **Start**.

Check that the volume was reused, not created new:
```bash
docker volume ls          # still vaultwarden_data, no new <othername>_data
```

### If you used a different folder name by mistake
If the folder is `/opt/stacks/passwords`, Compose makes a new empty volume `passwords_data`. Vaultwarden then says
"Invalid master password" because that account does not exist in the new volume. Your vault is still in
`vaultwarden_data`. To fix it:
1. Stacks > Remove > tick "Delete named volumes". Only the new, empty `passwords_data` is listed.
2. Click Remove.
3. Rename the folder to `vaultwarden`:
   ```bash
   sudo mv /opt/stacks/passwords /opt/stacks/vaultwarden
   ```
4. Adopt it again and Start.

Or keep the new name and point at the old volume:
```yaml
volumes:
  data:
    external: true
    name: vaultwarden_data
```

## 5. A container made with `docker run`: Uptime Kuma

1. Containers > uptime-kuma > **View details** > **Compose** tab > **Save as new stack** > **Create**.
   - Dockhand writes the file to `/opt/dockhand/stacks/<environment>/uptime-kuma/compose.yaml`.
   - The volume is marked `external: true`, so the data stays.
2. Containers > uptime-kuma > Remove > **Confirm**. Remove the old container BEFORE Start. Otherwise Start fails with
   `Conflict. The container name "/uptime-kuma" is already in use`.
3. Stacks > uptime-kuma > **Start**.

## 6. Updates instead of Watchtower

- Settings > Environments > Edit environment > **Updates**:
  - Turn Enable scheduled update check ON. The default is Daily 04:00, notify only. "Automatically update containers"
    is a separate switch.
  - Set the Timezone (the default is UTC).
- One container: Containers > Edit > Settings > Auto-update:
  - Turn Enable automatic image updates ON and choose Daily / Weekly / Custom (cron, for example `0 3 * * *`).
  - Dockhand pulls and recreates that one container. The rest of its stack (database, Redis) is left alone.
- Alerts:
  1. Settings > Notifications > **Add channel** > Type **Webhooks** > URL `ntfy://<server-ip>:<port>/<topic>` (plain
     http; `ntfys://` for https) > **Test** > **Add**.
  2. Settings > Environments > Edit > **Notifications** > add the channel. All 33 events are ON by default; switch off
     what you do not want.

## 7. Retire Watchtower and Portainer

```bash
docker stop watchtower && docker rm watchtower
docker stop portainer && docker rm portainer
docker volume rm portainer_data      # only after every editor-made stack has been copied out (step 4)
```
Do not use Portainer's stack Start/Update/Remove buttons on stacks Dockhand now runs.

## Note: containrrr/watchtower on Docker 29
The original image no longer works on current Docker Engine:
`Error response from daemon: client version 1.25 is too old. Minimum supported API version is 1.40`
(the repo was archived on 2025-12-17). The maintained fork `nickfedor/watchtower` works.

## Lab note: HTTPS for Vaultwarden and ntfy
The Vaultwarden web vault refuses a plain-http server address ("Insecure URL not allowed. All URLs must use HTTPS."), and
the ntfy web app needs https for browser notifications.

The lab uses a home CA and a certificate for the server IP:
```bash
sudo mkdir -p /root/labca /opt/certs && cd /root/labca
sudo openssl req -x509 -newkey rsa:2048 -nodes -keyout ca.key -out ca.crt -days 3650 -subj "/CN=Home Lab CA"
sudo openssl req -newkey rsa:2048 -nodes -keyout /opt/certs/homelab.key -out srv.csr -subj "/CN=<server-ip>"
printf "subjectAltName=IP:<server-ip>\nextendedKeyUsage=serverAuth\n" | sudo tee srv.ext
sudo openssl x509 -req -in srv.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out /opt/certs/homelab.crt -days 365 -extfile srv.ext
```
- Vaultwarden: `ROCKET_TLS: '{certs="/ssl/homelab.crt",key="/ssl/homelab.key"}'` plus volume `/opt/certs:/ssl:ro`, and
  `DOMAIN=https://<server-ip>:8081`.
- ntfy: `NTFY_LISTEN_HTTPS: ":443"`, `NTFY_CERT_FILE`, `NTFY_KEY_FILE`, plus port `8443:443`.
- Import `ca.crt` into the client's Trusted Root store.

A reverse proxy with a real certificate does the same job.
