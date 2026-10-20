# Corporate CA for an outbound proxy with TLS inspection

Place the PEM file of your corporate CA here (for example `corporate-ca.pem`) and set in `.env`:

    NODE_EXTRA_CA_CERTS=/etc/eviworx/proxy-ca/corporate-ca.pem

The folder is mounted read-only into the backend and all workers. Node ignores a missing
file without a warning — check the path after the restart:

    docker compose exec backend node -e "fetch('https://lic.eviworx.com').then(r => console.log(r.status))"

A TLS error (`self-signed certificate in certificate chain`) means the CA is not loaded.
