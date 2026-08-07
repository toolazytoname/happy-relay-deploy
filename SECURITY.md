# Security policy

The default deployment is loopback-only and reachable through a private
tailnet. Do not expose port 3005 directly.

At the verified revision, knowing a public relay address is sufficient for an
unrelated user to create a separate account space. A hidden hostname,
non-standard port, firewall logs, or public-key auditing does not provide
preventive access control. Public deployment therefore requires an explicit
risk decision and an independently reviewed access-control layer.

Keep Happy's master secret, SQLite database, files, TLS private keys, provider
tokens, and backups out of Git. Treat them as one recovery set. Run the relay
and coding agents as unprivileged users and pin upstream commits before every
upgrade.

Report vulnerabilities through GitHub's private security advisory feature.
