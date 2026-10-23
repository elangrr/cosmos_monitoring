# cosmos_monitoring

Monitoring for **Cosmos SDK nodes and validators**: Prometheus, Grafana and Alertmanager in one Docker Compose file, with a ready-made dashboard and Telegram alerts.

> This is the stack I use to monitor my own Cosmos SDK validators at [Indonode](https://indonode.net). I'm sharing it as-is so others can run the same setup. All server-specific values are removed: **you have to fill in your own** (node ports, server name, Telegram bot and chat).

## What you get

**Dashboard "Cosmos SDK nodes"** (one view for all your chains, with a chain filter):
- **Health:** block height, catching up or not, peers, validators in the active set.
- **Your validator:** blocks since your last signature, missed blocks (last 24h and per hour), your share of the voting power, voting power of offline validators on the network.
- **Consensus and performance:** blocks per minute, block time, consensus rounds, transactions per block, block processing time, mempool size and failed txs, P2P traffic, chain data size.
- **Server:** CPU, memory, disk.

**Alerts to Telegram:**

| Alert | When |
|---|---|
| CosmosNodeStalled | no new block for 15 min |
| CosmosMetricsDown | a node's metrics endpoint is unreachable for 5 min |
| CosmosNodeCatchingUp | node still catching up after 15 min |
| CosmosLowPeers | fewer than 3 peers for 15 min |
| ValidatorNotSigning | your validator has not signed for more than 10 blocks |
| ValidatorMissingBlocks | more than 20 missed blocks in 5 min |
| CosmosSlowConsensus | blocks need more than 2 rounds for 10 min |
| DiskFillingUp / DiskAlmostFull | disk full within 3 days / over 90% |
| PublicRpcDown | your public RPC gives no block height for 5 min (optional, needs rpc-prober) |
| PublicRpcBehind | your public RPC is more than 50 blocks behind other operators for 10 min (optional) |
| RpcProberStale | rpc-prober stopped writing results (optional) |

Everything uses the standard CometBFT metrics, so it works for any Cosmos SDK chain (CometBFT 0.37+). The validator panels only show data on chains where the node is a signing validator.

## Requirements

- Linux server with Docker and Docker Compose v2.
- Your Cosmos SDK node(s) running on the same server (or reachable from it).
- A Telegram bot token ([@BotFather](https://t.me/BotFather)) and your chat id, if you want alerts.

## Setup (about 10 minutes)

### 1. Turn on metrics in each node

In each node's `config.toml` (`~/.<daemon>/config/config.toml`), section `[instrumentation]`:

```toml
prometheus = true
prometheus_listen_addr = "127.0.0.1:26660"   # local only; use a different port per node
```

Restart the node, then check: `curl -s 127.0.0.1:26660/metrics | grep cometbft_consensus_height`

> Restarting a validator is your call. Do it at a quiet moment and watch it sign again.

### 2. Get the files

```bash
git clone https://github.com/elangrr/cosmos_monitoring.git
cd cosmos_monitoring
cp .env.example .env
```

### 3. Fill in your values

| File | What to change |
|---|---|
| `.env` | `GRAFANA_ADMIN_PASSWORD` (required). Ports only if the defaults clash with something on your server. |
| `prometheus/prometheus.yml` | Every `<PLACEHOLDER>`: one target per node (`127.0.0.1:<metrics port>`) with a `node` name, and your `<SERVER_NAME>`. |
| `alertmanager/alertmanager.yml` | `<YOUR_TELEGRAM_CHAT_ID>` (a number; group ids start with `-100`). |
| `secrets/telegram_bot_token` | Create it: `echo "<your bot token>" > secrets/telegram_bot_token && chmod 600 secrets/telegram_bot_token && sudo chown 65534:65534 secrets/telegram_bot_token` (Alertmanager runs as user `nobody`, 65534). Create this file **before** the first `docker compose up`, or Docker makes a folder in its place. |

Check the Prometheus config before starting:

```bash
docker run --rm -v "$PWD/prometheus:/etc/prometheus:ro" --entrypoint promtool prom/prometheus:v3.15.1 check config /etc/prometheus/prometheus.yml
```

### 4. Start

```bash
docker compose up -d
docker compose ps
```

### 5. Open Grafana

All services listen on `127.0.0.1` only, so nothing is exposed to the internet. From your own computer:

```bash
ssh -L 3000:127.0.0.1:3000 <user>@<your-server>
```

Then open http://127.0.0.1:3000, log in with the admin user and the password from `.env`, and open **Cosmos > Cosmos SDK nodes**.

### 6. Test an alert (optional)

```bash
curl -s -XPOST 127.0.0.1:9093/api/v2/alerts -H 'Content-Type: application/json' \
  -d '[{"labels":{"alertname":"TestAlert","severity":"warning"},"annotations":{"summary":"cosmos_monitoring test"}}]'
```

You should get a Telegram message within a minute.

## Adding a chain

Add one more target to the `cosmos` job in `prometheus/prometheus.yml`, then reload:

```bash
docker compose restart prometheus
```

The new chain appears in the dashboard's **Chain** filter by its `chain_id`.

## Watching your public RPCs (optional)

The node metrics above come from inside the node. They do not tell you whether your **public** RPC
(`https://rpc.<chain>.yourdomain.com`) works for users: a broken proxy, an expired certificate or a
DNS mistake all look fine from inside. [rpc-prober](https://github.com/elangrr/rpc-prober) checks the
public URL every minute, compares its block height with other operators' RPCs and hands the result to
Prometheus through node-exporter's textfile collector.

### 1. Install rpc-prober

Needs Rust (https://rustup.rs):

```bash
git clone https://github.com/elangrr/rpc-prober.git
cd rpc-prober && cargo build --release --locked
sudo install -m 755 target/release/rpc-prober /usr/local/bin/rpc-prober
```

### 2. Config

```bash
sudo mkdir -p /etc/rpc-prober
sudo cp rpc-prober/config.toml.example /etc/rpc-prober/config.toml   # from this repo
sudo nano /etc/rpc-prober/config.toml                               # your RPCs + 1-2 reference RPCs per chain
rpc-prober -c /etc/rpc-prober/config.toml                            # test: prints a table
```

Example output:

```
CHAIN     TYPE    ENDPOINT                                STATUS         HEIGHT  BEHIND  LATENCY  ERROR
atomone   cosmos  https://atomone-rpc.allinbits.services  ok           10930202       0   101 ms
atomone   cosmos  https://rpc.atomone.yourdomain.com      ok           10930202       0    75 ms
```

### 3. Run it every minute as its own user

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin rpcprober
sudo install -d -o rpcprober -g rpcprober -m 755 /var/lib/node_exporter/textfile
sudo cp rpc-prober/rpc-prober.service rpc-prober/rpc-prober.timer /etc/systemd/system/   # from this repo
sudo systemctl daemon-reload && sudo systemctl enable --now rpc-prober.timer
cat /var/lib/node_exporter/textfile/rpc.prom   # rpc_up, rpc_block_height, rpc_blocks_behind, rpc_latency_seconds
```

node-exporter in `docker-compose.yml` already reads `/var/lib/node_exporter/textfile`. If you run
node_exporter as a system service instead, add `--collector.textfile.directory=/var/lib/node_exporter/textfile`.

### 4. Turn on the alerts

In `prometheus/rules/rpc.yml` replace `yourdomain\\.com` with your domain (only your own endpoints
alert; the reference RPCs are only there for the comparison). Then:

```bash
docker compose restart node-exporter prometheus
```

To test: point one RPC in the config at a URL that does not exist under your domain; after 5 minutes
you get `PublicRpcDown` in Telegram, and a "resolved" message once you put it back.

## Ports (defaults, all on 127.0.0.1)

| Service | Port | Change with |
|---|---|---|
| Grafana | 3000 | `GRAFANA_PORT` |
| Prometheus | 9090 | `PROMETHEUS_PORT` (Grafana follows automatically; also update the `alerting` target only if you change Alertmanager) |
| Alertmanager | 9093 | `ALERTMANAGER_PORT` (also update `alerting` in `prometheus.yml`) |
| node-exporter | 9100 | `NODE_EXPORTER_PORT` (also update the `node` job in `prometheus.yml`) |

## Troubleshooting

- **Dashboard empty:** open http://127.0.0.1:9090/targets (through the tunnel). Every target should be `UP`. `DOWN` usually means metrics are off in `config.toml` or the port is wrong.
- **Validator panels empty:** normal if that node is not a signing validator.
- **Grafana does not start:** port 3000 is already used. Set `GRAFANA_PORT` in `.env`.
- **No Telegram alerts:** check `docker logs cm-alertmanager`; the bot must have been started (send it `/start`) or added to the group.

## Security notes

- Never commit `.env` or `secrets/` (both are in `.gitignore`).
- Keep the services on `127.0.0.1`. If you expose Grafana, put it behind HTTPS and a strong password.
- This stack only **reads** metrics. It never touches your keys or signs anything.

## License

MIT, see [LICENSE](LICENSE). Maintained by [Indonode](https://indonode.net).
