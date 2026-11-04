# Zammad on Wodby

What Wodby sets up for Zammad on this service. It runs the official `ghcr.io/zammad/zammad` image in four Deployments from the `wodby/zammad` chart:

| Workload | Process | Port |
| --- | --- | --- |
| `nginx` (primary) | Zammad's NGINX: serves `/assets`, sends `/ws` to `websocket`, everything else to `railsserver` | 8080, the main endpoint |
| `railsserver` | Puma, the Rails web app and ActionCable (`/cable`) | 3000, in-cluster only |
| `websocket` | the legacy WebSocket server used by the desktop app | 6042, in-cluster only |
| `scheduler` | the background worker; its init container installs or migrates Zammad | none |

Every workload runs one replica. The scheduler must never run twice, so it is recreated on each deploy.

## Install and upgrade

- On every deploy, the scheduler pod's init container runs Zammad's own `zammad-init`: it installs the database on a new app (migrations and seeds, no `CREATE DATABASE`) or runs migrations, clears the cache and syncs translations on later deploys. Then it applies the settings below and builds the Elasticsearch index when none exists.
- `nginx`, `railsserver` and `websocket` wait in the image's entrypoint until no migration is pending, so they never serve a half-migrated database. A new install takes a few minutes before the domain answers.
- Init output is in the scheduler pod's `init` container log. If it fails, the scheduler pod stays in `Init` and the deployment does not finish; fix the cause and deploy again. Init is safe to run again.
- Do not add a separate migration step or action.

## First administrator

- Zammad's getting started page lets whoever opens a new Zammad first create the administrator. Init creates the administrator before any web pod starts, so the page is never offered (`/api/v1/getting_started` answers 403).
- When no active administrator exists, init creates one with the roles Admin and Agent and access to all groups: sign-in and email from the required setting **Administrator email** (`ZAMMAD_ADMIN_EMAIL`), name from **Administrator first name** and **Administrator last name**, password from the generated token `admin_password` (`ZAMMAD_ADMIN_PASSWORD`). It marks the system set up and loads the English text modules.
- When an administrator exists, init creates nothing and changes no user. Changing the settings or the token later does not change an existing administrator; change it in Zammad under **Profile** or **Admin > Users**.
- When no administrator exists and the email already belongs to another user, or the password does not meet Zammad's password policy, init fails with Zammad's message and the deployment does not finish. Set another email and deploy again.
- While a Zammad import (OTRS, Zendesk, Freshdesk, Kayako) is running, init creates nobody. Imports from the getting started page are not available, because the system is already set up.
- The organization name, logo and email channels are set in Zammad after sign-in. Customer self-registration (**Register as a new customer**) is on by default in Zammad; turn it off under **Admin > Security > Base** if customers must not sign up.

## Settings Wodby manages

Init writes these Zammad settings on every deploy, so changes made in Zammad's admin pages are overwritten:

- `fqdn` from the service's primary domain (`ZAMMAD_FQDN`) and `http_type` `https`. Links in notifications and the allowed ActionCable origin follow it. After changing the primary domain, deploy again.
- `es_url`, `es_index` (`zammad`), `es_user` and `es_password` from the `elasticsearch` link, or an empty `es_url` without it. Values are read from the environment, so any character works.

## Linked services

| Link | Required | Variables |
| --- | --- | --- |
| `db`: PostgreSQL 15 or newer | yes | `DATABASE_URL` (with `pool=50`), `POSTGRESQL_HOST`, `POSTGRESQL_PORT` |
| `redis`: Redis | yes | `REDIS_URL` |
| `memcached`: Memcached | yes | `MEMCACHE_SERVERS` |
| `elasticsearch`: Elasticsearch 8.15 or newer, below 10 | no | `ELASTICSEARCH_HOST`, `ELASTICSEARCH_PORT`, `ELASTICSEARCH_SCHEMA`, `ELASTICSEARCH_USER`, `ELASTICSEARCH_PASS` |
| `sendmail`: mail transfer agent | no | `SMTP_HOST`, `SMTP_PORT` |

- `DATABASE_URL`, `REDIS_URL` and `ELASTICSEARCH_PASS` contain passwords; do not copy them into a repository.
- Each Zammad process opens up to 50 database connections; the stack raises PostgreSQL's `max_connections`.
- Without Elasticsearch, Zammad searches the database, which is slower and less complete. Set `ELASTICSEARCH_ENABLED=false` to turn it off while the link exists. OpenSearch is not supported by Zammad.
- With a `sendmail` link, init points Zammad's untouched notification channel (placeholder host `host.example.com`) at the relay, without TLS or authentication. A notification channel configured in Zammad is never changed. Incoming email and ticket email channels are configured in Zammad.

## Attachments

Zammad stores attachments in the database by default, so no volume is needed and all pods are stateless. Back up the PostgreSQL database. For S3-compatible storage, add `S3_URL` through the integration "Additional Zammad environment variables", deploy, then choose S3 under **Admin > System > Storage**. File storage is not supported, because the pods share no volume.

## Changing configuration

Add environment variables through the integration "Additional Zammad environment variables", then deploy. Every workload receives them. Useful ones: `TZ`, `ZAMMAD_WEB_CONCURRENCY`, `MAX_THREADS`, `ZAMMAD_PROCESS_DELAYED_JOBS_WORKERS`. `RAILS_TRUSTED_PROXIES` lists private ranges so Zammad logs the client address.

## Check the result

- `GET /favicon.ico` on port 8080 is NGINX's readiness check; `GET /` on port 3000 is Rails' readiness check. Liveness and startup checks are TCP.
- `/` opens the sign-in page. Sign in with **Administrator email** and the `admin_password` token, or check with `GET /api/v1/users/me` and basic auth.
- `/ws` and `/cable` answer `101` to a WebSocket upgrade.
