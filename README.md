# arobase.chat — bundler config

A ready-to-run Docker Compose stack that operates the off-chain services of an [arobase.chat](https://arobase.chat) instance on one machine:

| Service | Purpose |
|---|---|
| `caddy` | HTTPS for the bundlers (automatic Let's Encrypt), plus an internal RPC proxy with failover |
| `alto-gnosis`, `alto-arc`, `alto-base` | ERC-4337 bundlers ([Alto](https://github.com/pimlicolabs/alto) 0.0.21, EntryPoint v0.8, EIP-7702) for the sponsored chains |
| `relay` | Decentralized Web Push relay ([arobase-chat-push-relay](https://github.com/arobasechat/arobase-chat-push-relay)) |
| `monitor` | Health checks with webhook alerts: RPCs, paymaster deposits and budgets, bundlers, executor balances, relay activity |

Only ports 80 and 443 are exposed.

## Why run your own bundler?

The app sends sponsored user operations to the bundlers listed in the onchain `Registry`, in order, and falls back to the next one on failure. Running your own bundler removes the dependency on public bundlers, which stay listed as fallbacks. Gas is advanced by the bundler's executor and refunded right away by the EntryPoint from the paymaster's deposit, so an executor only needs a small float.

## RPC failover

Alto accepts a single RPC URL. Each bundler therefore talks to Caddy (`http://caddy:8101`, `:8102`, `:8103`), which proxies to the RPCs listed in `RPC_GNOSIS`, `RPC_ARC` and `RPC_BASE`, in order:
- every 15 s, Caddy sends a real `eth_chainId` call to each one;
- it takes out of rotation any RPC that fails, is too slow, or answers 5xx/429.

Transactions are broadcast to all RPCs in parallel (`ALTO_SEND_TRANSACTION_RPC_URL`).

## Setup

**1. DNS.** Point `bundler-gnosis`, `bundler-arc` and `bundler-base` (subdomains of `DOMAIN`) to the server, and open ports 80 and 443.

**2. Keys.** Create two keys per chain with `cast wallet new`:
- **executor**: sends the bundle transactions. Fund it with about 0.5 xDAI, 1 USDC on Arc, and 0.003 ETH on Base.
- **utility**: deploys Alto's simulation contract at startup and refills the executor. A few cents are enough.

**3. Configuration.**

```bash
cp .env.example .env
for c in gnosis arc base; do cp alto/$c.env.example alto/$c.env; done
chmod 600 .env alto/*.env
```

| File | What to set |
|---|---|
| `alto/<chain>.env` | `ALTO_EXECUTOR_PRIVATE_KEYS`, `ALTO_UTILITY_PRIVATE_KEY` |
| `.env` | `DOMAIN`, `ACME_EMAIL`, `REGISTRY` (Registry address), `REGISTRY_RPC`, `RPC_*` lists, `MONITOR_WEBHOOK`, `MONITOR_BALANCES` (executor addresses), `MONITOR_MIN_DEPOSITS` |

These files are ignored by git: never commit them. Shared, non-secret Alto settings live in `alto/common.env`.

**4. Push relay (once).**

```bash
mkdir -p relay-data && sudo chown 1000:1000 relay-data
docker compose run --rm relay pnpm relay init --network mainnet --contact mailto:you@example.org
RELAY_PK=0x… docker compose run --rm -e RELAY_PK relay pnpm relay register --network mainnet --name "My relay" --info "https://example.org"
```

`relay-data/` holds the relay's private keys and subscription state: **back it up and never share it**. `RELAY_PK` is the relay operator's address; it needs a few cents of xDAI.

**5. Start.**

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f alto-gnosis
```

The relay and monitor images are built from `RELAY_CONTEXT`, which must point to an exact commit (`https://github.com/arobasechat/arobase-chat-push-relay.git#<commit-sha>`) or to a local checkout (`../arobase-chat-push-relay`). These containers hold the relay's keys, so never build them from a moving branch: a compromised push would run on your server at the next build.

## Alto settings

| Setting | Why |
|---|---|
| `ALTO_SAFE_MODE` | `true` on Gnosis and Arc: the strict ERC-4337 validation rules (ERC-7562) run through `debug_traceCall` with a JavaScript tracer, so an operation whose validation depends on changing state is refused before it can cost the executor gas. It is `false` on Base because that node does not enable the JS tracer; there, the paymaster only sponsors the standard EIP-7702 account and caps gas and fees. Check a node with `debug_traceCall` and a JS tracer before turning it on. |
| `ALTO_SUPPORTS_EIP7623=true` | Prague's calldata floor is taken into account (it matters for image uploads). |
| `ALTO_CHAIN_TYPE=op-stack` | For Base: includes the L1 data fee in gas estimation. |
| `ALTO_ENABLE_CORS=true` | The browser calls the bundler directly. |

## Public endpoints

Caddy exposes only `POST` (and CORS preflight) on `bundler-*`. `/metrics` and `/health` answer 404 from outside, and each IP is limited to `RATE_LIMIT` requests per minute and per bundler (120 by default). The Caddy image is built with the [caddy-ratelimit](https://github.com/mholt/caddy-ratelimit) plugin, pinned to a commit in `caddy/Dockerfile`.

## Plugging the bundlers into the app

List your bundler first, with a public one as fallback. Do it either in `tooling/networks.mainnet.json` of [arobase-chat-evm-contracts](https://github.com/arobasechat/arobase-chat-evm-contracts) before deploying, or afterwards:

```bash
pnpm propose set-bundlers 100 https://bundler-gnosis.example.org https://public.pimlico.io/v2/100/rpc
```

## License

[Apache-2.0](LICENSE). Redistributions must keep the [NOTICE](NOTICE) file; please credit "arobase.chat" if you host a fork.
