# Ticket #439 — Test Results

Target: `ssh -i qa_gluu_org.pem ec2-user@184.34.206.32` (external), internal 172.31.39.68.
RHEL 9.8 "Plow", kernel 5.14.0-687.48.1.el9_8.x86_64, x86_64. Standard Flex **6.2.0**
(`flex-6.2.0-stable.el9.x86_64.rpm`) → Janssen **v2.2.0** (byte-verified against the shipped
RPM — see `RESEARCH/FLEX_6_2_0_JANSSEN_SOURCE_REVIEW.md`). Local persistence backend:
**PostgreSQL** (`-local-rdbm pgsql`, the release's documented default; confirmed by the RPM's
own dependency pull). Date: 2026-09-17.

Legend: PASS / FAIL / NOT TESTED. Evidence lives on the VM under
`/root/ticket-439-evidence/` (0700, root-only — sanitized JSON/cert artifacts only,
no tokens/passwords/secrets).

## Preflight

| Item | Result | Notes |
|---|---|---|
| SSH access | PASS | `ec2-user` + supplied key (not `ubuntu` as first given — owner corrected) |
| Sudo | PASS | Passwordless |
| OS identity | PASS | RHEL 9.8 confirmed via `/etc/os-release` + `hostnamectl` |
| Resource sizing | **CAUTION** | 2 vCPU / 7.5 GB RAM / 47 GB disk vs. official guideline of 4 vCPU / 8 GB RAM / 20 GB disk. Install completed successfully anyway (~7 min), but this is below the documented guideline — not re-verified under production-like load. |
| RHEL 9.8 vs. other RHEL9 minors | NOT TESTED | Docs state "RHEL 9" support generically; no minor-version statement found. Only 9.8 was exercised here — do not extrapolate to other 9.x minors. |
| SELinux | PASS (as Enforcing→Permissive) | Official docs require permissive; VM was Enforcing, switched (backed up original `/etc/selinux/config`) |
| firewalld / ports 80,443 | PASS | firewalld not installed on this AMI; nothing blocked 80/443 at OS or security-group level pre-install |
| DNS for testflexgen.gluu.info | **FAIL (environment)** | Does not resolve, internally or externally, at test time. Worked around with `curl --resolve` / `openssl -servername` (explicit client-side resolution, no `/etc/hosts` edits, no sudo needed on the test client) — labeled limitation throughout |
| Flex 6.2.0 el9 asset availability | PASS | Downloads correctly from GitHub release (1,368,556,603 bytes, sha256 `7f36c98c…0c7d3`), even though docs.gluu.org's current example now shows 6.4.1 |
| SSA | N/A (resolved) | Owner-supplied SSA verified as a well-formed, unexpired RS256 JWT but confirmed by source (both a dev checkout and the exact `v6.2.0` tag) to be unused by the CLI installer (`ssa=''` hardcoded); relevant only to the untested Admin UI web upload step |

## Install (stock, `-host-name testflexgen.gluu.info`)

| Item | Result | Notes |
|---|---|---|
| RPM install | PASS | `dnf install ./flex-6.2.0-stable.el9.x86_64.rpm`, pulled PostgreSQL + httpd + mod_ssl etc. |
| `flex_setup.py` with CLI flags (`-host-name` etc.) | **FAIL — packaging bug** | Crashes: `error: argument -h/--help: ignored explicit argument 'ost-name'`. Source-confirmed in both this checkout and the `v6.2.0` tag: `flex_setup.py`'s own `get_flex_setup_parser().parse_known_args()` runs against full `sys.argv` before forwarding anything, and doesn't register `-host-name` itself, so Python argparse's short-option "attached value" heuristic collides with the built-in `-h/--help`. Reproduces on a stock, unmodified release — unrelated to hostname preservation. |
| Install via `setup.properties` + `-n -f <file>` | PASS | Workaround for the above: `-f`/`-n` don't collide with `-h`. Confirmed `Config` attribute names from `setup_app/config.py`: `hostname, orgName, countryCode, city, state, admin_email, encode_salt, admin_password, rdbm_type, rdbm_password` |
| Install completion | PASS | "Janssen Server installation successful!" + "Installation was completed." Admin UI installed. ~7 minutes total. |
| Secret handling | PASS | `admin_password`/`rdbm_password`/`encode_salt` generated via `openssl rand`, stored 0600 root-only, never printed |

## Baseline (before OS hostname reset)

| Item | Result | Notes |
|---|---|---|
| OS hostname after stock install | **Confirms ticket hypothesis** | `hostnamectl`/`hostname`/`hostname -f`/`/etc/hostname` all became `testflexgen.gluu.info` — the stock installer overwrites the OS static hostname with the service FQDN on RHEL9 via `hostnamectl set-hostname`, unconditionally, exactly as Pollen's source review predicted |
| `/etc/hosts` | PASS | Installer added `172.31.39.68 testflexgen.gluu.info testflexgen`; no admin-hostname entry existed to conflict with |
| Discovery (`/.well-known/openid-configuration`) | PASS | HTTP 200, `issuer: https://testflexgen.gluu.info` |
| JWKS | PASS | HTTP 200, 22 keys |
| TLS certificate | PASS (with caveat) | Self-signed, `CN=testflexgen.gluu.info`, subject==issuer, SHA256 fingerprint `E1:E0:8D:43:0F:9D:38:15:56:45:EB:3A:80:32:FD:BC:BD:AC:B3:B9:45:21:89:CA:6E:D7:0F:DC:52:7F:AB:27`. Validated with genuine trust (`curl --cacert`/`openssl -CAfile` against the actual server cert, **not** `-k`/insecure) — `Verify return code: 0 (ok)`. **Caveat**: the cert has **no SAN extension** (CN-only). This validated with curl/OpenSSL here, but modern browsers (Chrome/Firefox, since ~2017) reject CN-only certs outright regardless of trust — expect browser TLS errors even after trusting this exact cert, unless a SAN is added. |
| Client credentials grant | PASS | HTTP 200, `token_type: Bearer`, access token issued |
| Token issuance | PASS | Same as above |
| Introspection (authorized client) | PASS | HTTP 200, `active: true`, `iss: https://testflexgen.gluu.info` |
| Browser authorization-code + PKCE, ID-token validation | **NOT TESTED** | No browser capability in this environment. `/authorize` and `/token` endpoints exist and discovery advertises `code`/PKCE support, but an actual interactive login was not exercised — not claiming this passed. |
| Admin UI login | **NOT TESTED** | No browser capability. HTTP reachability of `/admin` was not treated as a login test. |
| Dedicated least-privilege test client | **PARTIAL** | Attempted to provision a fresh minimal OIDC client via the Config API; the only readily available authenticated client (`jca`, the auto-provisioned Config API/Admin UI role-based client) is itself scoped to Admin-UI session-management scopes only and does **not** carry `clients.write` — creating a new client via CLI was not achievable within that scope. Used the `jca` client itself (already a narrowly-scoped, non-admin client) for the client_credentials/introspection tests above instead of a purpose-built one. |

## Post-install OS hostname reset (`hostnamectl set-hostname ip-172-31-39-68.us-west-2.compute.internal`)

| Item | Result | Notes |
|---|---|---|
| Reset via `hostnamectl` | PASS | `hostname`, `hostname -f`, `/etc/hostname` all became the admin FQDN |
| `/etc/hosts` service-FQDN entry preserved | PASS | Untouched — matches source finding that the rewrite only strips lines matching the *service* FQDN token |
| Admin hostname still resolves | PASS | `getent hosts` resolves (via VPC resolver) |
| Service restart (httpd, jans-auth, jans-config-api, jans-fido2, jans-scim) | PASS | All active, no errors in `jans-auth` journal |
| Discovery/JWKS/cert/issuer unchanged after reset | PASS | Byte-identical to baseline; issuer still `https://testflexgen.gluu.info` |
| Client credentials + introspection after reset | PASS | Same as baseline |
| **Reboot** | PASS | VM back up ~20s after `sudo reboot` |
| Hostname persists as admin FQDN after reboot | PASS | `hostnamectl`/`hostname -f`/`/etc/hostname` all correct |
| **cloud-init interaction (new finding)** | — | `cc_update_hostname` runs with `frequency=always` on **every boot** and independently reasserts the AWS-metadata-derived hostname (`ip-172-31-39-68.us-west-2.compute.internal`), regardless of `preserve_hostname: false` in `/etc/cloud/cloud.cfg`. This happened to match our target admin hostname here. **Caveat**: I manually reset the hostname via `hostnamectl` before rebooting in this run, so I did not isolate "stock install leaves it as `testflexgen.gluu.info` → reboot alone, with no manual step → does cloud-init silently revert it?" as a separate clean test. The log evidence (`cc_update_hostname.py[DEBUG]: Updating hostname to ip-172-31-39-68...`) strongly suggests it would, but this is inferred from the always-frequency mechanism, not independently isolated. Do **not** treat cloud-init's own behavior as a substitute for the preserve-os-hostname patch below — it only affects the state *after* a reboot, not the (potentially long) live window immediately following install, and depends on cloud-init's cached metadata matching the desired admin hostname, which won't hold in every environment. |
| Services healthy after reboot | PASS | All 5 services + PostgreSQL active within ~90s of boot (this box is under-provisioned; expect slower JVM startup than on the recommended spec) |
| Post-reboot discovery/JWKS/cert/issuer | PASS | Byte-identical to baseline |
| Post-reboot client credentials + introspection | PASS | Same as baseline |
| New TLS/DNS/database/init errors in logs | PASS (none found) | Scanned httpd/jans-auth/jans-config-api/jans-fido2/jans-scim journals since reboot |

## Strict requirement: never change the OS hostname at all (Step 5)

| Item | Result | Notes |
|---|---|---|
| Native preserve-OS-hostname flag exists? | **FAIL (confirmed absent)** | Checked the full ~90-option CLI surface in `arg_parser.py` at the exact `v2.2.0` tag. No such option. |
| Custom patch produced | PASS | `preserve-os-hostname.patch` — adds `--preserve-os-hostname` CLI flag / `preserve_os_hostname=True` properties-file key, threaded through `arg_parser.py` → `setup_options.py` → `Config` → guards only the `hostnamectl`/`/bin/hostname`/`/etc/hostname`/`/etc/sysconfig/network` calls inside `update_hostname()`. The `/etc/hosts` block (which makes the service FQDN locally resolvable) and `Config.hostname` itself (certs/issuer/Apache) are completely untouched by the guard. |
| Patch applies cleanly | PASS | `patch -p1 --dry-run` and real apply both succeed against the pristine, v2.2.0-identical source (verified via SHA256 of the pre-patch files, recorded in the patch header) |
| Patched files compile | PASS | `python3 -m py_compile` on all 4 touched files |
| Component-level logic test | PASS | Isolated mock of `update_hostname()`'s control flow confirms: `preserve_os_hostname=False` → `hostnamectl` is called (stock behavior intact); `preserve_os_hostname=True` → no OS-identity-mutating call is made. This is a logic-level test, not a live install. |
| **Fresh end-to-end preserve-hostname installation** | **NOT TESTED** | Would require either wiping this VM's current working install (not authorized — the ticket explicitly says do not reimage or wipe the first working installation) or a second clean VM (none available in this session). This is the one requirement from the brief not executed end-to-end; everything else needed to attempt it (patch, install command shape, properties-file mechanism) is ready and evidenced above. |

## Summary

Everything that could be tested on this single VM without wiping the working install or
requiring a browser: **PASS**. Two items are explicitly **NOT TESTED** and reported as such
rather than assumed: (1) interactive browser flows (auth-code+PKCE login, Admin UI login),
(2) a fresh from-scratch install with the preserve-os-hostname patch applied. The patch itself
is built, applies cleanly, compiles, and passes a component-level logic test.
