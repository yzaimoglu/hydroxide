# Running hydroxide over Tailscale

This guide covers running hydroxide on a home/server box and reaching it from
your other devices over [Tailscale](https://tailscale.com). Tailscale gives you
a private, WireGuard-encrypted network, so hydroxide's plaintext protocols are
safe to use without adding TLS.

## 1. Log in to ProtonMail

On the server, authenticate once. This prints a **bridge password** — the
password your mail client uses (not your ProtonMail password). It is shown only
once and is not stored anywhere, so copy it now.

```shell
hydroxide auth <protonmail-username>
```

Your ProtonMail credentials are stored on disk encrypted with this bridge
password.

## 2. Find the server's Tailscale IP

```shell
tailscale ip -4        # e.g. 100.101.102.103
```

By default hydroxide binds every server to `127.0.0.1` (loopback only), which
is unreachable over Tailscale. You must bind it to the Tailscale IP.

> Bind to the Tailscale IP specifically, **not** `0.0.0.0`. `0.0.0.0` would also
> expose the plaintext ports to your local LAN. The Tailscale IP restricts
> access to devices on your tailnet.

## 3. Run the bridge

`serve` runs SMTP, IMAP and CardDAV together:

| Protocol | Default port |
|----------|--------------|
| SMTP     | 1025         |
| IMAP     | 1143         |
| CardDAV  | 8080         |

```shell
hydroxide \
  -smtp-host    100.101.102.103 \
  -imap-host    100.101.102.103 \
  -carddav-host 100.101.102.103 \
  serve
```

Run only one protocol instead of `serve` if you prefer: `hydroxide imap`,
`hydroxide smtp`, `hydroxide carddav`.

## 4. Configure your mail client

Point the client at the server's Tailscale IP (or its MagicDNS name):

* **Hostname:** `100.101.102.103` (or `your-host.your-tailnet.ts.net`)
* **IMAP port:** 1143 &nbsp;·&nbsp; **SMTP port:** 1025
* **Connection security:** None
* **Authentication:** Normal / plain password
* **Username:** your ProtonMail username
* **Password:** the bridge password from step 1

## Do I need TLS?

**No.** Tailscale already encrypts all traffic end-to-end with WireGuard, so
running IMAP/SMTP in plaintext inside the tunnel is fine — adding TLS would just
double-encrypt. When no certificate is configured, hydroxide allows plaintext
authentication, so this works out of the box.

Add TLS only if:

* your mail client refuses plaintext auth (some mobile clients do), or
* you don't fully trust every device on your tailnet.

Tailscale can issue a real Let's Encrypt certificate for your MagicDNS name:

```shell
tailscale cert your-host.your-tailnet.ts.net

hydroxide \
  -imap-host your-host.your-tailnet.ts.net \
  -tls-cert  your-host.your-tailnet.ts.net.crt \
  -tls-key   your-host.your-tailnet.ts.net.key \
  serve
```

Then set your client's connection security to SSL/TLS.

## IMAP caveats worth knowing

IMAP support is work-in-progress. It works for reading, searching, flagging and
moving mail, but:

* **Creating / renaming / deleting folders from the client is not supported.**
  Manage folders and labels in the ProtonMail web UI instead — they appear
  automatically.
* **UIDs are not stable across re-syncs.** Opening a mailbox re-downloads its
  message index, and clients that cache aggressively (e.g. Thunderbird, mobile)
  may occasionally re-download or reindex. Prefer syncing the Inbox and specific
  folders over the huge *All Mail* folder.
* Custom ProtonMail labels show up as IMAP keywords/flags, not folders.

## Keeping it running

Run hydroxide under a process supervisor so it restarts on reboot — for example
a systemd user service, or `tmux`/`screen` for a quick setup. Set the bridge
password non-interactively via the `HYDROXIDE_BRIDGE_PASS` environment variable.
