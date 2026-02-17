# Deploying the Cocoon Client on Railway

Use this to run the **Cocoon client** on [Railway](https://railway.app) (or any PaaS that sets `PORT` and env vars). The client lets your app request AI generations from the Cocoon network and pay in TON. You are **not** running workers or proxies—only the client that connects to the existing network.

## Requirements

- **OWNER_ADDRESS** – Your TON wallet address (receives change, appears in contracts).
- **NODE_WALLET_KEY** – Wallet private key (base64) for the client. Keep it secret; set in Railway as a secret variable.
- **TON config** – Either ship `spec/mainnet-full-ton-config.json` in the image (see [Deployment](#deployment)), or set **TON_CONFIG_BASE** and **TON_CONFIG** to paths inside the container.

Optional:

- **PORT** – Set by Railway automatically. The client HTTP API listens on this port.
- **COCOON_BIN_DIR** – Directory containing pre-built `router`, `client-runner`, and `cocoon-subst`. If set, the launch script skips building and uses these binaries.
- **ROOT_CONTRACT_ADDRESS** – Override the Cocoon root contract (default is production).

## Using the launch script

From a clone with the repo root as working directory:

```bash
# Railway sets PORT and often RAILWAY=1
export OWNER_ADDRESS="UQ..."
export NODE_WALLET_KEY="base64..."
# Optional if spec/ is in the image:
# export TON_CONFIG_BASE=/app/spec/mainnet-base-ton-config.json
# export TON_CONFIG=/app/spec/mainnet-full-ton-config.json

./scripts/cocoon-launch --railway
```

Or with a config file that sets `type = client` and the same vars:

```bash
./scripts/cocoon-launch --railway client.conf
```

With **pre-built binaries** (e.g. in CI or a Docker image):

```bash
export COCOON_BIN_DIR=/app/bin
export OWNER_ADDRESS="..."
export NODE_WALLET_KEY="..."
./scripts/cocoon-launch --railway
```

The script will:

1. Use **PORT** for the client HTTP server (Railway’s public port).
2. Read **OWNER_ADDRESS**, **NODE_WALLET_KEY**, and optional **ROOT_CONTRACT_ADDRESS**, **TON_CONFIG_BASE**, **TON_CONFIG** from the environment when `--railway` is used.
3. Use **COCOON_BIN_DIR** if set to run `router` and `client-runner` without building.

## Deployment

### Option A: Build on deploy (Nixpacks / Railway default)

If Railway builds from source:

1. Root of the repo must be the project root (so `scripts/cocoon-launch` and `spec/` exist).
2. Set env: **OWNER_ADDRESS**, **NODE_WALLET_KEY** (and optionally **ROOT_CONTRACT_ADDRESS**, **TON_CONFIG_BASE**, **TON_CONFIG**).
3. Set **RAILWAY=1** (or use `--railway` in the start command).
4. Start command:

   ```bash
   python3 scripts/cocoon-launch --railway
   ```

   The first run will run CMake and build the client binaries; subsequent runs use the same build.

Ensure `spec/mainnet-base-ton-config.json` exists in the repo. Add `spec/mainnet-full-ton-config.json` (with liteservers) if you have it; otherwise the script will fail when it tries to copy the full TON config. You can obtain a full TON config from the TON documentation or run a node.

### Option B: Docker with pre-built binaries

Use a multi-stage Dockerfile that builds the client (and optionally router + cocoon-subst), then runs with **COCOON_BIN_DIR** and **--railway**:

- **Build stage:** Clone repo, submodules, `cmake` + `ninja` target `cocoon-all`, then copy `build/client-runner`, `build/tee/router`, `build/tee/cocoon-subst` into a single directory (e.g. `/app/bin`).
- **Run stage:** Copy that directory, the `scripts/` and `spec/` trees, set **COCOON_BIN_DIR=/app/bin**, **RAILWAY=1**, and **CMD** to `python3 scripts/cocoon-launch --railway`.

Then set **OWNER_ADDRESS** and **NODE_WALLET_KEY** (and any TON config paths) in Railway.

## Client API

Once the client is running, it exposes an HTTP server on **PORT**:

- **OpenAI-compatible** inference endpoint: send POST requests (e.g. `/v1/chat/completions`-style) to the client; it forwards them to Cocoon proxies and returns the model response.
- **/stats**, **/jsonstats** – client statistics.
- **/request/topup**, **/request/charge**, **/request/withdraw**, **/request/close** – manage balance and stake with proxies (see [smart contracts](smart-contracts.md)).

Your app (hosted on Railway or elsewhere) should call this client’s URL (e.g. `https://your-service.railway.app`) for inference and use the same TON wallet for payments.

## Notes

- Only **client** mode is supported on Railway. Workers and proxies require Intel TDX and (for workers) NVIDIA H100+ GPUs and cannot run on Railway.
- Keep **NODE_WALLET_KEY** secret and fund the wallet with TON so the client can pay for requests.
- For production, provide a real **mainnet-full-ton-config.json** (with liteservers) so the client can talk to the TON blockchain.
