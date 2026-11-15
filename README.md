# Minecraft Server Management Suite
This is a suite of apps that makes managing a Minecraft server on a Raspberry Pi or a Mini PC easier for both the server admin and the players on the server. The suite includes a Server Manager App, a Player Access App, and API services for your Minecraft server.

This management suite is built upon the Raspberry Pi Minecraft documentation and was tested using a NeoForge Minecraft server running on a Raspberry Pi 5. 

## Prerequisites
* [NeoForge Minecraft Server](https://github.com/alxthedesigner/raspberry-pi-5-minecraft-server/blob/main/README.md)
* Raspberry Pi 5
* Minecraft 26.3 
* NeoForge 26.3.0.60-beta / 26.3.0.76 
* Tailscale VPN (for server access. Players and the server admin must be connected to [Tailscale](https://tailscale.com)) to reach the server.
* `git clone` this repo

  There are 3 parts of this suite consisting of 2 apps and an API:

  | Part | Runs from | Description | Source |
  |---|---|---|---|
  | **Manager API Services**| Raspberry Pi or Mini PC server; Port 8080 | FastAPI backend services that allows the server manager app to interact with the server | `manager-api/`|
  | **Server Manager App**| Desktop or Command line | Dashboard to interact with the server; start/stop the server, send console commands over RCON, add or remove mods from the server, access the whitelist and config files, and streams the server log over WebSocket. | `server-manager-app/` |
  | **Player Access App** | Raspberry Pi or Mini PC server; Port 5000 | Small Flask website for players see instructions on how to join the server, the server status, the current mods on the server available to download, and players online. Only accessible when player is on the Tailscale VPN (or whichever VPN you use). | `player-access-app/` |


---

# Part 1. Manager API Services (on Raspberry Pi)

1. Enable RCON in `server.properties`
RCON (remote control) lets you send commands to the Minecraft server, like whitelisting, changing game modes for users, etc.
Never reuse this password anywhere else, and never commit `server.properties` file.

  ```properties
  enable-rcon=true
  rcon.port=25575
  rcon.password=choose-a-long-random-password
  ```

2. Configure the Minecraft Server
  Copy `.env.example` to `.env` and fill in real values (`.env` is git-ignored):
  
  - `MC_SERVER_DIR` — your Minecraft server folder, e.g. `/home/pi/minecraft_server`
  - `RCON_PASSWORD` — must match `rcon.password`
  - `RCON_PORT` — must match `rcon.port` (default 25575)
  - `API_KEY` — a long random string. **Every API request must send it** (`X-API-Key` header), so set the same value in the admin app's Settings and in the player portal's `.env`. Generate one with `python3 -c "import secrets; print(secrets.token_urlsafe(32))"`.
  - `ALLOWED_ORIGINS` — the player portal's address
  
    Edit `minecraft-manager.service` and replace the example user `pi` and `/home/pi/...` paths with your own username and home folder.


3. Install Minecraft server as a service
Install the Minecraft server and start the server on boot (of the Raspberry Pi) 

```bash
sudo cp minecraft-manager.service /etc/systemd/system/
sudo mkdir -p /etc/systemd/system/minecraft-manager.service.d
sudo cp minecraft-manager.service.d/killmode.conf /etc/systemd/system/minecraft-manager.service.d/
sudo systemctl daemon-reload
sudo systemctl enable --now minecraft-manager
sudo systemctl status minecraft-manager
```

  `killmode.conf` sets `KillMode=process`. Without it, restarting the API also kills a Minecraft server that was started from the app, because systemd treats the server as part of the API service.


4. How the server uses the API
Start command:

```bash
tmux new-session -d -s minecraft -c "$MC_SERVER_DIR" './run.sh nogui'
```


Memory (`user_jvm_args.txt`):

`run.sh` reads Java settings from `user_jvm_args.txt` in the server folder. Out of the box every line is a comment, so Java only gets a quarter of the Pi's RAM. Add a line such as:

```
-Xmx6G
```

Leave roughly 1 GB for the operating system (on a 2 GB board use about `-Xmx1300M`).

---


# Part 2. Server Manager App (on your Desktop)

1. Install app packages
```bash
cd server-manager/admin-app
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python main.py
```

2. Features and Main Functions
   
| Feature | Function |
|---|---|
| **Dashboard** | Server status, Start / Stop / Restart / Wake, **Backup World**, Pi CPU, memory, swap, disk and temperature (°C and °F), non-whitelisted join alerts, **Player Portal on/off** |
| **Console** | Live server log (last 200 lines on connect, follows the log across server restarts), send server commands, optional filter for RCON noise |
| **Players** | Online players, whitelist editor, approve/deny whitelist requests from the portal |
| **Mods** | Group mods into **packages**, tick which mods in each package are active, upload many `.jar` files at once, and switch the server to a package in one click |
| **Config** | Edit `server.properties`, JVM arguments and NeoForge/mod config files |
| **Instructions** | The player portal's "How to Join" steps (read from the portal's page, so they always match), as a refresher |
| **Activate Package on Server** | enables the checked mods in that package and moves every other mod into `mods_disabled/`. Nothing is deleted. Restart the server afterwards.|
| **Download Packages…** | downloads a zip of one or more mods (called packages) that a player adds to their `mods` folder. Packages are stored in the Server Manager App's settings and are published to the Raspberry Pi's `mod_packages.json` file |
| **Backup World** | downloads the whole world as minecraft_world_backup_<date>_<time>.zip to the Desktop running the Server Manager App. On the Raspberry Pi, it turns autosave off (save-off), writes everything to disk (save-all flush), zips the world, and always turns autosave back on. The download is written to a .part file |



3. On first launch, open **Settings** (bottom of the sidebar) and enter your Pi's Tailscale IP or MagicDNS name, SSH username, the server folder on the Pi, API port `8080`, the RCON password and the API key. These are saved only on your computer.

4. Add Desktop Application of **Minecraft Server Manager**:

```bash
./install_mac_app.sh
```
This adds **Minecraft Server Manager** to `/Applications`. Logs: `~/Library/Logs/Minecraft Server Manager.log`.


---

# Part 3. Player Access App (on Raspberry Pi)
Players can see server status and who's online, download the **active mod package** (or tick several packages and download them together), and follow the join steps. Downloads go through this player access app to the API, so the API key never reaches players' browsers.

1. Install and configure the Player access App on the raspberry pi

```bash
cd ~/minecraft-manager/player-portal
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env    # set API_KEY to the same value as the API, and a random FLASK_SECRET
```

2. Edit `player-portal.service` with your own username and paths, then:

```bash
sudo cp player-portal.service /etc/systemd/system/minecraft-player-portal.service
sudo systemctl daemon-reload
```

3. The Player Access App deliberately does **not** start at boot. Turn it on and off from the Manager App (**Player Portal → Turn On / Turn Off**). The API does this with `sudo systemctl start|stop minecraft-player-portal.service`, so give the API's user permission for exactly those two commands (replace `pi` with your username):

```bash
echo 'pi ALL=(root) NOPASSWD: /usr/bin/systemctl start minecraft-player-portal.service, /usr/bin/systemctl stop minecraft-player-portal.service' \
  | sudo tee /etc/sudoers.d/minecraft-player-portal
sudo chmod 440 /etc/sudoers.d/minecraft-player-portal
sudo visudo -c
```

  Only run one player access service. An older or second copy bound to `0.0.0.0:5000` would expose the portal to your whole LAN and fight over the port.
  The portal listens on `127.0.0.1:5000`. To share it with players over your tailnet only, use `tailscale serve` on the Pi.


---
# Security and Troubleshooting

### Security notes

- Keep the API, RCON and portal reachable only over your LAN/tailnet. Do not port-forward 8080, 5000 or 25575.
- Set `white-list=true` and `online-mode=true` in `server.properties`.
- Never commit `.env`, `server.properties`, `whitelist.json`, `ops.json`, TLS certificates or keys, or your world folder. The repository's `.gitignore` excludes them.


### Troubleshooting

**Admin app can't connect** — check the Pi's Tailscale IP (`tailscale ip`), that `sudo systemctl status minecraft-manager` is running, and that the API key matches.

**Console is empty** — on connect the API sends the last 200 real log lines (RCON status-check lines are skipped, and if today's log is quiet it fills in from the previous log). If nothing appears, check the API is running, the API key in Settings matches, and the WebSocket (`ws://<pi>:8080/ws/log`) is reachable.

**Server won't start from the app** — check `tmux` is installed, `MC_SERVER_DIR` is correct and contains `run.sh`, and run `./run.sh nogui` by hand to see the error. A common cause after an upgrade is a missing required library mod; the error is near the top of `logs/latest.log`.

