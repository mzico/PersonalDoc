**Status: this is not an officially supported Gluu/Janssen configuration.** No vendor
confirmation was sought or given. Everything below reflects what was independently observed
on one RHEL 9.8 VM.

## Tested configuration

- **Flex**: 6.2.0, RHEL9 asset (`flex-6.2.0-stable.el9.x86_64.rpm`)
- **Underlying Janssen**: v2.2.0 (byte-verified — the RPM ships the same
  `config.py`/`installers/jans.py`/`utils/arg_parser.py`/Apache templates as the upstream
  `v2.2.0` tag, not `main`)
- **OS**: Red Hat Enterprise Linux **9.8** ("Plow"), x86_64, kernel `5.14.0-687.48.1.el9_8`.
  Only this minor version was exercised — the vendor docs state "RHEL 9" support generically
  with no minor-version statement, so do not assume other 9.x minors behave identically.
- **Persistence backend**: local PostgreSQL (`-local-rdbm pgsql`) — this is the release's
  documented default and what its RPM dependencies pull in. No other backend was tested.
- **Resources**: 2 vCPU / 7.5 GB RAM / 47 GB disk. The official VM guideline is 4 vCPU / 8 GB
  RAM / 20 GB disk. Install and all tests below completed successfully on the smaller spec,
  but this was not validated under production-like load — match the guideline for anything
  beyond a lab.

## Prerequisites

- EPEL: `sudo dnf -y install "https://dl.fedoraproject.org/pub/epel/epel-release-latest-$(rpm -E %rhel).noarch.rpm"`
- `sudo dnf -y install mod_auth_openidc`
- SELinux: official docs require **permissive** mode, not enforcing. Back up
  `/etc/selinux/config` first if you need to reason about drift later.
- firewalld: allow HTTPS if firewalld is active on your image (`sudo firewall-cmd
  --permanent --zone=public --add-service=https && sudo firewall-cmd --reload`). Note: the
  RHUI-based AWS RHEL9 AMI used for this test doesn't ship firewalld at all — check your own
  image rather than assuming either way.
- DNS: point your public FQDN at the VM's external IP *before* install if you want a clean
  first-pass TLS/discovery test. If DNS isn't ready yet, you can validate the install with
  explicit client-side resolution instead of editing `/etc/hosts` or waiting on DNS:
  `curl --resolve <fqdn>:443:<external-ip> --cacert <server-cert> https://<fqdn>/...`
  or `openssl s_client -connect <external-ip>:443 -servername <fqdn> -CAfile <server-cert>`.
  Label this clearly as a lab-only workaround in your own notes — it is not a substitute for
  real DNS in production.

## Known packaging issue: don't pass installer fields as CLI flags to `flex_setup.py`

On this exact release, running:
```
sudo python3 /opt/jans/jans-setup/flex/flex-linux-setup/flex_setup.py -host-name your.fqdn ...
```
**crashes** with `error: argument -h/--help: ignored explicit argument 'ost-name'`. This is a
real packaging bug, and specifically a **`flex-linux-setup` wrapper bug, not a Janssen/`jans`
core issue** — pinned to `flex-linux-setup/flex_linux_setup/flex_setup.py`'s own
`get_flex_setup_parser()`, invoked before `jans` is ever touched. That wrapper parser never
registers `-host-name` itself (only `jans`'s own `arg_parser.py` does, later, and `-host-name`
works fine there standalone), so Python's argparse misreads it as a malformed use of the
default `-h`/`--help` option and exits before the intended value is ever forwarded downstream.
Confirmed in source, reproduces on a stock unmodified install.

**Workaround**: use a `setup.properties` file with `-f` and `-n` instead of CLI flags for the
installer fields:
```
cat > /path/to/setup.properties <<PROPS
hostname=your.fqdn
orgName=Your Org
countryCode=US
city=YourCity
state=YourState
admin_email=admin@your.fqdn
encode_salt=<24-char-random-string>
admin_password=<generated-secret>
rdbm_type=pgsql
rdbm_password=<generated-secret>
PROPS
chmod 600 /path/to/setup.properties

sudo python3 /opt/jans/jans-setup/flex/flex-linux-setup/flex_setup.py \
  -n -f /path/to/setup.properties \
  --flex-non-interactive --install-admin-ui --no-progress
```
Generate `encode_salt`/`admin_password`/`rdbm_password` with something like `openssl rand`,
store them in a 0600 file, and never commit or print them.

## SSA / license

An SSA (Software Statement Assertion) is documented as required to complete Flex setup and
activate Admin UI licensing. In practice (confirmed by source at the `v6.2.0` tag), the CLI
installer sets its internal SSA field to empty regardless of anything you pass — the SSA is
only consumed later via the **Admin UI web upload** (`https://<fqdn>/admin`). Have a valid
SSA ready before you need Admin UI licensing, but it will not block or change the CLI install
itself.

## What happens to the OS hostname on install

**On RHEL 9 with the systemd + RPM install path, the stock installer overwrites your OS
static hostname with the service FQDN, unconditionally, via `hostnamectl set-hostname`.**

Two ways to handle this, depending on your actual requirement:

### Option A — Post-install restoration (OS hostname change is temporary/acceptable)

If your requirement is "the OS hostname must end up correct, temporarily changing during
install is fine":
```
sudo hostnamectl set-hostname your-admin-hostname.internal
```
This is a native, OS-supported command — no patching needed. Verified on this release:
- The application's `/etc/hosts` entry for the *service* FQDN is untouched by this — the
  installer's own `/etc/hosts` rewrite logic only strips lines matching the service FQDN as a
  token, never the OS hostname, so both names keep resolving locally without conflict.
- Issuer, certificate identity, JWKS, and all tested OIDC flows are completely unaffected —
  they're all driven by `Config.hostname` (the service FQDN baked in at install time), not by
  the live OS hostname.
- This was verified to survive a service restart (httpd + all Janssen services) and a full
  reboot.
- **AWS/cloud-init caveat**: on an EC2 instance managed by cloud-init, `cc_update_hostname`
  runs on **every boot** (not just first boot) and independently reasserts whatever
  AWS instance metadata says the hostname should be, regardless of the
  `preserve_hostname` setting in `/etc/cloud/cloud.cfg`. In this test that happened to
  align with the desired admin hostname, effectively "fixing itself" on reboot even without
  the manual command above. **Do not rely on this** — it depends on your instance metadata
  matching your desired hostname, it only helps after a reboot (not during the window between
  install and next reboot), and it won't apply outside EC2/cloud-init-managed hosts at all.

### Option B — Strict preservation (OS hostname must never change, not even briefly)

No supported flag exists for this in Flex 6.2.0 / Janssen v2.2.0. Use the enclosed
**custom patch**, `preserve-os-hostname.patch` (not an upstream option — clearly a local
modification, review before use in any environment you don't fully control):
- Adds a `--preserve-os-hostname` flag (and equivalent `preserve_os_hostname=True`
  properties-file key) that guards only the OS-identity-mutating calls inside
  `update_hostname()` — `hostnamectl`/`/bin/hostname`/`/etc/hostname`/
  `/etc/sysconfig/network`. It does not touch the `/etc/hosts` block that makes your service
  FQDN resolvable, and does not touch `Config.hostname` (certs/issuer/Apache ServerName) at
  all — your service FQDN configuration is identical either way.
- Verified: applies cleanly against the pristine v2.2.0-identical source (checksums in the
  patch header), all 4 touched files compile, and an isolated component-level test confirms
  the guard logic behaves correctly in both the on and off state.
- **Not yet verified**: a full fresh install with this patch applied, end to end, on a clean
  VM. That would require either wiping the currently-working test VM or provisioning a second
  one — neither was available/authorized in this session. Do this validation before relying
  on the patch for anything beyond a component-level review.

## TLS certificate note

The installer generates a self-signed certificate using `CN=<service FQDN>` with **no SAN
(Subject Alternative Name) extension**. This validated fine with `curl`/`openssl` when
explicitly trusting that exact certificate, but modern browsers (Chrome, Firefox, since
around 2017) reject CN-only certificates outright regardless of trust store — expect a
browser TLS error on `/admin` even after installing this cert as a trusted CA, unless you
replace it with a properly SAN-bearing certificate (e.g., via your own CA or a public CA).

## Verification checklist

```
# Discovery + issuer
curl --resolve <fqdn>:443:<ip> --cacert <server-cert> https://<fqdn>/.well-known/openid-configuration

# JWKS
curl --resolve <fqdn>:443:<ip> --cacert <server-cert> https://<fqdn>/jans-auth/restv1/jwks

# TLS identity
openssl s_client -connect <ip>:443 -servername <fqdn> -CAfile <server-cert>
```
Confirm `issuer` is `https://<fqdn>` (your service FQDN, not the OS hostname) in all cases,
before and after any hostname change.

