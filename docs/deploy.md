# Deploy

Nilo on Arc is one Docker image: the Hono server answers `/api`, serves the web app and runs the scheduled agent.
CI builds it on every push to `main` as `ghcr.io/nearx-labs/nilo-arc:latest` (Arc Testnet by default; set the
repository variable `ARC_NETWORK=mainnet` for a mainnet bundle).

It runs on the same server as Nilo (Stellar), behind the same Traefik, at `arc.nilo-ai.xyz`.

## 1. DNS

An `A` record for `arc.nilo-ai.xyz` pointing at the server (the same IP as `nilo-ai.xyz`).

## 2. Environment file

`/etc/nearx/nilo-arc.env`, readable only by root (`chmod 600`):

```bash
ARC_NETWORK=testnet
OPENAI_API_KEY=...        # the same key Nilo uses
OPENAI_MODEL=gpt-5.4-mini
NILO_OPERATOR_KEY=0x...   # the operator in packages/core/src/config.ts; the server refuses any other
```

The operator key can only call `NiloAgent.invest` (inside each user's mandate) and relay gasless claims. Keep a
few USDC on it for gas.

## 3. Compose service

Add next to the `nilo` service in `/etc/nearx/docker-compose.yml`. Use the same entrypoint, certificate resolver
and network names as the `nilo` service's labels.

```yaml
  nilo-arc:
    image: ghcr.io/nearx-labs/nilo-arc:latest
    restart: unless-stopped
    env_file: nilo-arc.env
    volumes:
      - nilo-arc-data:/app/data   # the agent's activity log and the list of wallets it watches
    labels:
      - traefik.enable=true
      - traefik.http.routers.nilo-arc.rule=Host(`arc.nilo-ai.xyz`)
      - traefik.http.routers.nilo-arc.entrypoints=websecure
      - traefik.http.routers.nilo-arc.tls.certresolver=letsencrypt
      - traefik.http.services.nilo-arc.loadbalancer.server.port=5173
```

and under the top-level `volumes:`:

```yaml
  nilo-arc-data:
```

## 4. Start it

```bash
cd /etc/nearx && sudo docker compose pull nilo-arc && sudo docker compose up -d nilo-arc
```

`docker compose logs nilo-arc` should print `Nilo server on http://localhost:5173 (Arc Testnet)`. If the pull is
refused, the server's GHCR login can't read the new package: make `nilo-arc` public in the organization's
packages, or log in with a token that has `read:packages`.

## 5. Deploy on every push (optional)

Add the same `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY` and `SSH_PASSPHRASE` secrets as `Nearx-Labs/nilo` to this
repository. The workflow's deploy job is skipped until `SSH_HOST` exists.

## Mainnet

Fill the mainnet addresses in [`packages/core/src/config.ts`](../packages/core/src/config.ts), set the repository
variable `ARC_NETWORK=mainnet` (the web bundle) and `ARC_NETWORK=mainnet` in `nilo-arc.env` (the server), then
push.
