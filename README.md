See the overview of the project at https://attestation.app/about.

## Installation guide

This is a generic guide on setting up the attestation server.

Sync submodules for sqlite4java prebuilt in libs/.

You need to set up nginx using the nginx configuration in the nginx directory in this repository.
You'll need to adjust it based on your domain name. The sample configuration relies on certbot,
[nginx-rotate-session-ticket-keys](https://github.com/GrapheneOS/nginx-rotate-session-ticket-keys),
and [certbot-ocsp-fetcher](https://github.com/tomwassenberg/certbot-ocsp-fetcher). Setting up the web server is out-of-scope for this guide.

The deploy and process scripts on the development machine require the following dependencies to be installed:

    rsync, zopfli, parallel, yajl, brotli, nginx-mod-brotli, python3, python-pip, nodejs, npm, libxml2

`validatornu` must be obtained manually
[here](https://github.com/validator/validator/releases/tag/20.6.30) (vnu.linux.zip). Unzip it in
the same directory in AttestationServer and change the binary path in `process-static` from
`validatornu` to `vnu-runtime-image/bin/vnu` using sed: `sed -i
's+validatornu+vnu-runtime-image/bin/vnu+g' process-static`

Install a headless Java 18 runtime environment. The package name on Debian-based distributions is
`openjdk-18-jre-headless` or `jre-openjdk-headless` on Arch Linux. Install `sqlite3` in order to
set up the email configuration for the database.

As root, on the server:

    useradd -m -s /bin/bash -b /var/lib attestation

    mkdir -p /opt/attestation/deploy_{a,b}
    ln -s /opt/attestation/deploy_a /opt/attestation/deploy

    mkdir -p /srv/attestation.app_{a,b}
    ln -s /srv/attestation.app_a /srv/attestation.app

Set up ssh `authorized_keys` for the attestation user.

Copy `attestation.service` to `/etc/systemd/system/attestation.service`.

On your development machine, you will need to change the `remote` variable in the deploy scripts
to your server. You'll also need to change the
[DOMAIN](https://github.com/GrapheneOS/AttestationServer/blob/df53e39706fdc6247ce497cb0776bf77fe931085/src/main/java/app/attestation/server/AttestationServer.java#L85)
in AttestationServer.java to your server and the [app signing
key](https://github.com/GrapheneOS/AttestationServer/blob/df53e39706fdc6247ce497cb0776bf77fe931085/src/main/java/app/attestation/server/AttestationProtocol.java#L157-L162)
and [app
ID](https://github.com/GrapheneOS/AttestationServer/blob/df53e39706fdc6247ce497cb0776bf77fe931085/src/main/java/app/attestation/server/AttestationProtocol.java#L154-L156).
Then deploy the attestation server and static content:

    npm i
    python -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt
    ./deploy-server
    ./deploy-static

As root on the server, enable and start the attestation server:

    systemctl enable attestation
    systemctl start attestation

The server will be listening on `[::1]:8080` by default which [can be changed](https://github.com/GrapheneOS/AttestationServer/blob/df53e39706fdc6247ce497cb0776bf77fe931085/src/main/java/app/attestation/server/AttestationServer.java#L320).

## Email alert configuration

In order to send email alerts, AttestationServer needs to be configured with valid credentials for
an SMTP server. The configuration is stored in the `Configuration` table in the database and can
be safely modified while the server is running to have it kick in for the next email alert cycle.

Only SMTPS (SMTP over TLS) with a valid certificate is supported for remote email servers.
STARTTLS is deliberately not supported because it's less secure unless encrypted is enforced, in
which case it makes more sense to use SMTPS anyway. The username must also be the full address for
sending emails.

For example, making an initial configuration:

    sqlite3 attestation.db "INSERT INTO Configuration VALUES ('emailUsername', 'alert@attestation.app'), ('emailPassword', '<password>'), ('emailHost', 'mail.grapheneos.org'), ('emailPort', '465')"

The `attestation.service` unit only allows the service to communicate over `localhost` by default
so the `IPAddressDeny`/`IPAddressAllow` configuration either needs to be removed or extended to
include your DNS server and mail server IP addresses when using a remote mail server.

### Handling abuse

The `emailBlacklistPatterns` array in
[`src/main/java/app/attestation/server/AttestationServer.java`](https://github.com/GrapheneOS/AttestationServer/blob/df53e39706fdc6247ce497cb0776bf77fe931085/src/main/java/app/attestation/server/AttestationServer.java#L92-L94) can be used to blacklist email
addresses using regular expressions. We plan to move this to a table in the database so that it
can be configured dynamically without modifying the sources, rebuilding and redeploying. For now,
this was added to quickly provide a way to counter abuse.

## API for the Auditor app

### QR code

The scanned QR code contains space-separated values in plain-text: `<domain> <userId>
<subscribeKey> <verifyInterval>`. The `subscribeKey` should be treated as an opaque string rather
than assuming base64 encoding. Additional fields may be added in the future.

### /auditor/challenge

* Request method: POST
* Request headers: n/a
* Request body: n/a
* Response body:

Returns a standard challenge message in the same format as the Auditor app QR code. The challenge
can only be used once and expires in 1 minute.

The server challenge index is always zeroed out and the userId should be used instead.

### /auditor/verify

* Request method: POST
* Request headers:

The `Authorization` header needs to be set to `Auditor <userId> <subscribeKey>` for an unpaired
attestation. That will also work for a paired attestation if the subscribeKey matches, but it
should be set to `Auditor <userId>` to allow for subscribeKey rotation.

* Request body:

Standard attestation message in the same format as the Auditor app QR code.

* Response body:

Returns JSON with a `verifyInterval` field.

## Logging

Logs are written via stderr for journald. Log guidelines:

emerg: filesystem or database corruption
alert: service fully not functioning
crit: service partially not functioning
error: service side error for a specific request
warning: login failures, unexpected errors triggered by clients and missing setup for full functionality such as sending alerts
info: security-relevant events such as logins, expected errors triggered by clients and status logging
debug: not used in production
