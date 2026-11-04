# Zammad service for Kubernetes on Wodby

Run Zammad as a reusable Kubernetes application service with Wodby.

This repository defines the Wodby service manifest and operational
configuration for Zammad.

- [Browse Wodby services](https://wodby.com/services)
- [Wodby service documentation](https://wodby.com/docs/2.0/services/)
- [Service manifest reference](https://wodby.com/docs/2.0/services/template/)

## Wodby stacks using this service

- [Zammad application stack](https://github.com/wodby/stack-zammad)

## Service overview

| Property | Manifest configuration |
| --- | --- |
| Service name | `zammad` |
| Type | Application service |
| Versions | `7.2` by default |
| Workloads | `nginx` (Deployment, primary), `railsserver`, `websocket`, `scheduler` (Deployments) |
| Containers | one per workload, all using `ghcr.io/zammad/zammad` |
| Endpoints | `zammad`: HTTP 8080 (main) |
| Service links | PostgreSQL (`db`), required; Redis (`redis`), required; Memcached (`memcached`), required; Elasticsearch (`elasticsearch`), optional; Mail transfer agent (`sendmail`), optional |
| Application build | Not buildable from application source |
| Helm | chart `oci://registry-1.docker.io/wodby/zammad`; version `0.2.1` |
| Configuration and operations | 3 settings, 1 integration slot, 0 volumes, 0 actions |

## Use this service

Use this service through [Zammad application stack](https://github.com/wodby/stack-zammad), or reference `zammad` from a
custom Wodby stack.

A service is a reusable component and does not deploy by itself. The stack
defines its links, settings, versions, resources, and relationship to the rest
of the application.

## Runtime contract

Zammad runs as four processes from the official image: NGINX on the main
endpoint, the Rails web server, the WebSocket server and the background worker.
PostgreSQL 15 or newer, Redis and Memcached are required. Elasticsearch 8.15 or
newer (below 10) is optional and recommended for search. SMTP is optional and
used for notifications.

On every deploy, Zammad's own init installs the database on a new app or runs
its migrations, before the background worker starts. The web processes wait
until the database is current. Attachments are stored in the database, so no
volume is needed.

## First administrator

Wodby creates the first administrator before Zammad serves its first request, so
Zammad's getting started page is never offered to visitors. Sign in at the app's
main domain with **Administrator email** and the `admin_password` token from
**Configuration > Tokens**.

**Administrator email** is a required setting. This step creates nothing when an
administrator already exists. See `GUIDANCE.md`.

## Maintain a custom version

1. Fork this repository.
2. Edit the service manifest and referenced files.
3. Import the repository as a [Git-backed service](https://wodby.com/docs/2.0/services/create/#create-a-git-backed-service).
4. Reference the service from a stack manifest.

Keep service, workload, container, endpoint, link, volume, config, and
derivative names stable unless dependent stacks and app-level overrides are
updated at the same time.

Validate the manifests with:

```bash
wodby service validate-manifest service.yml --include GUIDANCE.md --org <org-id>
```

See the [service manifest reference](https://wodby.com/docs/2.0/services/template/) and the [managed services index](https://github.com/wodby/services).
