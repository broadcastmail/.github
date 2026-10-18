# BroadcastMail

Broadcast email with Supabase apps. Connect your database, send to your users.

## Repos

| Repo | Description |
|---|---|
| [broadcastmail-frontend](https://github.com/broadcastmail/broadcastmail-frontend) (private) | Next.js 16 frontend |
| [broadcastmail-api](https://github.com/broadcastmail/brodcastmail-api) | Spring Boot HTTP API |
| [broadcastmail-worker](https://github.com/broadcastmail/broadcastmail-worker) | Async campaign processor |
| [broadcastmail-webhooks](https://github.com/broadcastmail/broadcastmail-webhooks) | Stripe and Resend webhooks |
| [broadcastmail-common](https://github.com/broadcastmail/broadcastmail-common) | Shared domain library |

## Architecture

Modular monolith, four services, one PostgreSQL database. 
[→ See how it's built](../ARCHITECTURE.md)

## Links

[Website](https://broadcastmail.com) · [Contributing](../CONTRIBUTING.md)
