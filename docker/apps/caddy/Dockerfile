# Caddy with the Cloudflare DNS plugin, so it can solve ACME DNS-01 challenges
# and obtain real Let's Encrypt certs without opening any inbound ports.
# Pinned to the same Caddy version as the rest of the stack.
FROM caddy:2.11.4-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/cloudflare

FROM caddy:2.11.4-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
