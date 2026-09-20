# Deployment

The app runs on the home server (`<server-lan-ip>`, Tailscale `<server-tailscale-ip>`) in Docker,
and redeploys itself when `main` moves.

## Access

| | |
|---|---|
| **Preferred** (real TLS, from anywhere on the tailnet) | https://<server>.<tailnet>.ts.net:9443 |
| LAN, no Tailscale needed | http://<server-lan-ip>:8321 |

The HTTPS address is served by `tailscale serve` with a genuine Let's Encrypt
certificate. Prefer it — Safari upgrades bare `http://` addresses to `https://`,
which fails on the plain LAN URL.

Add the app to the iPhone Home Screen from the **HTTPS** address: a PWA is bound
to its origin, so one added from the `http://` URL stays on that origin.

## How auto-deploy works

The server holds a git clone at `~/apps/wardrobe`. A cron job runs every 5 minutes:

```
*/5 * * * * $HOME/apps/deploy-wardrobe.sh
```

`deploy-wardrobe.sh` fetches `origin/main` and compares it to the local `HEAD`.
If they match it exits silently. If they differ it runs `git reset --hard origin/main`
followed by `docker compose up -d --build`.

So the deploy flow is just:

```bash
git push        # from the Mac
# within 5 minutes the server is running the new code
```

Pull-based rather than a webhook, because the server has no public inbound
address — it sits behind Tailscale, so GitHub could not reach it to push an event.

## Data safety

`data/` is gitignored, which is what makes `git reset --hard` safe here: the
wardrobe database and uploaded photos are untracked, so a deploy never touches
them. **Nothing in `data/` is backed up by git** — it exists only on the server.

Back it up separately:

```bash
scp <user>@<server-lan-ip>:~/apps/wardrobe/data/wardrobe.db ./wardrobe-backup.db
```

## Access to the private repo

The server authenticates with a read-only SSH deploy key (`~/.ssh/wardrobe_deploy`),
bound to `github.com` in `~/.ssh/config`. Read-only means a compromised server
cannot push to the repo.

## Operations

```bash
cd ~/apps/wardrobe
docker compose logs -f          # watch the app
docker compose restart
cat ~/apps/wardrobe-deploy.log  # deploy history
./../deploy-wardrobe.sh         # force a deploy now
```

## Environment

AI features need `ANTHROPIC_API_KEY`. It is currently unset, so those features are
inactive and everything else works. To enable, create `~/apps/wardrobe/.env`:

```
ANTHROPIC_API_KEY=sk-ant-...
```

then `docker compose up -d`. `.env` is gitignored.
