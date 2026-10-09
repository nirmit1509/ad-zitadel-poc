# AD ↔ ZITADEL LDAP Federation POC

A disposable local setup proving that Active Directory can federate into
ZITADEL via LDAP, so AD users can log in through ZITADEL using their
existing AD credentials. Everything runs in Docker and is torn down when
you're done (`docker compose down -v`) — nothing touches any real AD or the
hosted ZITADEL instance the actual app uses.

Follow this guide top to bottom, in order — the certificate steps (3–5) are
mandatory setup, not a fix for something broken. Skipping them produces a
TLS error later.

## Prerequisites

- Docker Desktop running
- `ldapsearch` available locally (macOS: `brew install openldap`, or skip
  the verification commands — they're optional sanity checks)

## 1. Project files

This directory should already contain `docker-compose.yml`. It defines:

- **samba-ad** (`nowsci/samba-domain`) — disposable AD domain controller for
  `EXAMPLE.LOCAL`, LDAP on 389, LDAPS on 636. Runs `privileged: true` with a
  named volume (`samba-data`) because the domain provisioning needs to set
  POSIX ACLs that don't work on a plain overlay mount.
- **zitadel-db** (`postgres:16`) — backing store for ZITADEL.
- **zitadel** (`ghcr.io/zitadel/zitadel:v2.65.1`) — console at
  `http://localhost:8080/ui/console`. Pinned to `v2.65.1`: newer images
  (`latest`, v3/v4) split the login UI into a separate `ghcr.io/zitadel/login`
  service not bundled here — `latest` 404s on `/ui/v2/login/login` without it.
  Mounts `./zitadel-ca-bundle.crt` (built in step 5) over
  `/etc/ssl/certs/ca-certificates.crt` so it trusts Samba's cert — this
  file **must exist before `zitadel` starts**, which is why it comes up last.

## 2. Start the AD and the database

```bash
docker compose up -d samba-ad zitadel-db
```

Wait ~20s for the AD domain to finish provisioning:

```bash
docker logs samba-ad --tail 10
```

You're looking for `samba entered RUNNING state` with no crash loop.
(`samba_dnsupdate: ERROR: Record already exist` lines are harmless re-run
noise — ignore them.)

## 3. Why the certificate needs fixing

Samba auto-generates a self-signed LDAPS cert with **no Subject Alternative
Name** — only a bare `CN=DC1.example.local`. Modern TLS clients (including
ZITADEL, written in Go) stopped trusting the CN field for hostname matching
years ago and require a SAN entry matching the hostname being connected to.
With zero SAN entries, verification fails for *any* hostname:
`tls: failed to verify certificate: x509: certificate is not valid for any
names`. Steps 4–5 replace it with a cert that has the right SAN and make
ZITADEL trust it, before ZITADEL ever tries to connect — so you won't hit
that error at all.

(This is an artifact of this specific disposable test image's quick
self-signed cert generation. A real production AD domain controller
typically gets a properly SAN-issued cert via AD Certificate Services, so
you're unlikely to see this particular failure against a real AD — see
"Notes for a real AD" at the bottom.)

## 4. Generate a proper cert for samba-ad

Generate it **inside** the container (as root, via `docker exec`) rather
than copying from the host — that sidesteps host/container UID mismatches
that make Samba refuse to start (`invalid ownership of file ... CVE-2013-4476`).

```bash
docker exec samba-ad openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout /var/lib/samba/private/tls/key.pem \
  -out /var/lib/samba/private/tls/cert.pem \
  -days 365 \
  -subj "/CN=DC1.example.local" \
  -addext "subjectAltName=DNS:samba-ad,DNS:dc1.example.local,DNS:example.local"

docker exec samba-ad cp /var/lib/samba/private/tls/cert.pem /var/lib/samba/private/tls/ca.pem
docker exec samba-ad chmod 600 /var/lib/samba/private/tls/key.pem

docker restart samba-ad
```

Verify the new cert is live:

```bash
echo | openssl s_client -connect localhost:636 -servername samba-ad 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

Expect `DNS:samba-ad, DNS:dc1.example.local, DNS:example.local`.

## 5. Make ZITADEL trust that cert

ZITADEL (Go binary, scratch-based image, no shell) reads its system CA
trust from `/etc/ssl/certs/ca-certificates.crt`. Build an extended bundle —
the image's original trust store plus Samba's cert appended — using a
throwaway container so you don't need `zitadel` running yet:

```bash
docker create --name zitadel-cert-extract ghcr.io/zitadel/zitadel:v2.65.1 >/dev/null
docker cp zitadel-cert-extract:/etc/ssl/certs/ca-certificates.crt ./zitadel-ca-bundle.crt
docker rm zitadel-cert-extract >/dev/null

docker cp samba-ad:/var/lib/samba/private/tls/cert.pem /tmp/samba-cert.pem
echo "" >> ./zitadel-ca-bundle.crt
cat /tmp/samba-cert.pem >> ./zitadel-ca-bundle.crt
```

`docker-compose.yml` already bind-mounts this file into the `zitadel`
service, so it'll pick it up automatically on first start.

## 6. Start ZITADEL

```bash
docker compose up -d zitadel
```

Wait ~15s, then confirm it's healthy:

```bash
curl -s http://localhost:8080/debug/healthz   # expect: ok
```

Console: `http://localhost:8080/ui/console`, login `admin@example.local` /
`Passw0rd!Admin`.

## 7. Create a test AD user

```bash
docker exec samba-ad samba-tool user create jdoe 'Passw0rd!' \
  --given-name=Jane --surname=Doe --mail-address=jdoe@example.local

docker exec samba-ad samba-tool user list
docker exec samba-ad samba-tool user show jdoe
```

There's no web console for Samba AD — it's CLI/LDAP only, same as real AD.
Use `samba-tool` for anything else (more users, groups, etc.), or connect a
desktop LDAP browser (e.g. Apache Directory Studio) to `ldaps://localhost:636`
with bind DN `administrator@example.local` / `Passw0rd!Admin`, base DN
`dc=example,dc=local`, accepting the self-signed cert.

Sanity-check the AD side directly (Samba rejects plaintext simple binds on
389 — LDAPS only):

```bash
LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://localhost:636 \
  -D "jdoe@example.local" -w 'Passw0rd!' \
  -b "dc=example,dc=local" "(sAMAccountName=jdoe)" mail displayName
```

Expect `result: 0 Success` with `mail: jdoe@example.local` and
`displayName: Jane Doe`.

## 8. Wire ZITADEL to the AD (LDAP Identity Provider)

In the console: **Organization → Settings → Identity Providers → Add LDAP
Provider**.

| Field | Value |
|---|---|
| Servers | `ldaps://samba-ad:636` |
| Base DN | `dc=example,dc=local` |
| Bind DN | `CN=Administrator,CN=Users,DC=example,DC=local` |
| Bind Password | `Passw0rd!Admin` |
| User Object Classes | `user` |
| User Filters | `sAMAccountName` — **attribute name only**. Entering a full filter string like `(sAMAccountName=%s)` gets double-wrapped by ZITADEL's own filter construction and fails with `Filter Compile Error: finished compiling filter with extra at end`. |
| ID Attribute | `sAMAccountName` |
| Email Attribute | `mail` |
| Display Name Attribute | `displayName` |
| First Name Attribute | `givenName` |
| Last Name Attribute | `sn` |

Save, then activate the provider in **Login Policy** so it appears on the
login screen.

## 9. Test the federated login

Open the login page, pick the LDAP/SambaAD option, authenticate as `jdoe` /
`Passw0rd!`.

First login shows **"External User Not Found"** with the AD-sourced profile
(Given name: Jane, Family name: Doe, Email: jdoe@example.local) pre-filled —
this is expected, not an error; it confirms the LDAP bind, filter, and
attribute mapping all worked. Click **Register** to auto-provision the
ZITADEL user from those attributes. The account is created immediately.

ZITADEL will also try to send an email-verification message — this will
fail silently (`could not create email channel` in `docker logs zitadel`)
since no SMTP provider is configured here. That's expected and doesn't
block anything: the account already exists and is linked to the AD
identity. To clear the "unverified" state, use the "Set as verified" action
on the user's E-mail section in the console (admins can do this without a
code), or wire up a local mail catcher like Mailpit if you want to see the
actual email.

Confirm success: the user shows up under **Users** with the right email/name
and linked identity provider. Log out and back in as `jdoe` again — it
should re-authenticate the same user, not create a duplicate.

## Teardown

```bash
docker compose down -v
```

Removes all containers, volumes, and the network — nothing persists.

## Notes for a real AD

- **SAN issue (steps 3–5, part 1):** unlikely to recur. Real production
  domain controllers typically get a cert issued through AD Certificate
  Services (Domain Controller / Kerberos Authentication template), which
  includes a correct SAN by default. This was specific to the disposable
  test image's quick self-signed cert.
- **CA trust (steps 3–5, part 2):** will still apply, in a cleaner form.
  Enterprise DCs are signed by an internal/enterprise CA, not a public one,
  so ZITADEL still needs that CA trusted. Do it the proper way — upload the
  CA cert via ZITADEL's **Root CA** field on the LDAP IDP config (available
  through the Management API / Terraform provider's `root_ca` attribute,
  and present in newer console builds than the v2.65.1 pinned here) — rather
  than the OS-trust-store workaround used in step 5, which exists only
  because this console build didn't expose that field.
