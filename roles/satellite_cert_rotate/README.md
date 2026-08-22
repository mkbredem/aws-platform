# satellite_cert_rotate

Rotate or renew **custom** TLS certificates on Red Hat Satellite Server and
Capsule Servers, using the documented tooling: `katello-certs-check`,
`capsule-certs-generate`, and `satellite-installer`.

---

## NOT SUPPORTED - BY ANYONE

**This exists to illustrate what automating this procedure could look like.
It is not a product, not a deliverable, and not supported by anyone.**

- Not a Red Hat product. Not supported by Red Hat, Red Hat Support, or any Red
  Hat employee in any capacity - the author of this role included.
- No warranty, no service-level commitment, no maintenance, no bug fixes, and
  no commitment to answer questions about it, from Red Hat or from the author.
- **Not** validated against every Satellite version, topology, or
  configuration.
- Nothing here modifies Red Hat's support policy for a Satellite deployment,
  and running it does not entitle anyone to support for its behavior or for a
  deployment it has touched.
- Anyone who runs it does so entirely at their own risk and is responsible for
  reviewing it, testing it in a lab first, and deciding whether it fits their
  environment.

It wraps the procedure in *Administering Red Hat Satellite* → *Renewing
certificates*, read against the 6.16, 6.18, and 6.19 documentation. Where this
role and the product documentation disagree, **the documentation is correct**.

---

## Caveats you need to read before running this

**You will take an outage on each server.** `satellite-installer` restarts
Satellite services - httpd, foreman, dynflow, pulpcore, and the rest. While it
runs, the web UI, content delivery (yum/dnf clients pulling from Satellite or a
Capsule), subscription operations, and remote execution are unavailable.
Typical runs are 10-30 minutes per server; large deployments take longer. The
role rotates the Satellite first, then Capsules one at a time, so the total
window is the sum, not the max.

**Run this from somewhere that does not depend on the Satellite.** A workstation
or an Ansible Automation Platform cluster whose execution environments do not
pull content through the Satellite being rotated. If your execution nodes get
their packages or their trust store from this Satellite, rotating it mid-job can
strand the job.

**Changing the CA is the dangerous part, not changing the certificate.** If the
new certificate is issued by a different CA than the one currently in use, every
registered host, every Capsule, and every integration that pins the old CA
loses its connection to Satellite the moment the new one is deployed. The role
detects this by comparing the issuer of the new certificate with the issuer of
the certificate the host currently serves, and **refuses to run** unless you set
`satellite_cert_rotate_accept_ca_change=true`. Before you set that flag, deploy
the new CA to your hosts - Satellite's own guidance for the self-signed CA is a
phased dual-CA approach (add the new CA alongside the old, roll it out, then
rotate, then remove the old), and the same logic applies to a custom CA. See
"Refreshing the self-signed CA certificate on hosts" in *Managing hosts*, which
offers Script REX, Ansible REX, and manual methods. This role deliberately does
**not** touch registered hosts.

**Nothing is idempotent here.** `satellite-installer` re-runs its whole Puppet
catalog and restarts services every time. The role's mitigation is a
serial-number check: if the host already serves the certificate you are
deploying, it skips the host entirely
(`satellite_cert_rotate_skip_if_current`, on by default). Use
`satellite_cert_rotate_validate_only=true` for a dry run - it runs
`katello-certs-check` and stops. There is no meaningful `--check` support.

**Do not delete the certificate files afterwards.** The documentation is
explicit: Satellite needs the certificate files (and the Capsule needs its
certificate archive) at upgrade time. The role stages them into
`/root/satellite_cert_rotate` and `/root/capsule_certs` and leaves them there.

**The Capsule certificate archive contains that Capsule's private key.** It is
generated on the Satellite, relayed through the control node (or the execution
environment), and written to the Capsule as `0600` root-owned. If that relay is
not acceptable in your environment, run the Capsule play from a control node
inside the trust boundary, or move the archive out of band.

**Backups.** The role tars `/etc/pki/katello`, `/etc/foreman/ssl`, and
`/root/ssl-build` before the installer runs. That is a certificate rollback aid,
**not** a Satellite backup. Against any deployment that matters, a
`satellite-maintain backup` should be taken first, and it takes hours rather
than minutes.

**Rollback is another outage.** Re-run the role with the previous certificate,
key, and CA bundle. If the CA changed and hosts were updated, rolling back means
reverting the CA on those hosts too.

**Out of scope:** CSR generation and issuance (the certificate must already
carry X.509 v3 `Key Usage` and `Extended Key Usage` extensions with a CN
matching the server FQDN); renewal of the built-in **self-signed** CA, which is
a different, phased procedure; the Puppet CA, which is separate from the web
certificate; and anything on registered hosts.

---

## Notes on the manual procedure this was built from

Three things worth correcting or knowing about the manual steps this was built
from:

1. **The Capsule deployment command needs `--foreman-proxy-foreman-base-url`.**
   The 6.16, 6.17, 6.18, and 6.19 documentation all include it:

   ```
   satellite-installer --scenario capsule \
   --certs-tar-file "/root/My_Certificates/capsule.example.com-certs.tar" \
   --certs-update-server \
   --foreman-proxy-foreman-base-url "https://satellite.example.com"
   ```

   The flag that goes away in 6.18 is `--foreman-proxy-register-in-foreman
   "true"`, which 6.16 and 6.17 still document. The role adds it automatically
   when `satellite_cert_rotate_version` is below 6.18.

2. **"Run the unique command returned by `capsule-certs-generate`" is a trap for
   automation.** For a renewal on an already-registered Capsule the command is
   deterministic - it is the four flags above. The "unique" wording matters at
   *initial* Capsule setup, where the generated command carries one-time
   registration values. Scraping the installer command out of stdout and
   executing it is fragile across releases and is a code-execution path from
   command output. This role builds the command from the documented flags and
   saves the `capsule-certs-generate` output next to the archive so an operator
   can diff the two. If they diverge, the output from `capsule-certs-generate`
   is authoritative and the command this role builds should not be used.

3. **`katello-certs-check` lives on the Satellite, and its exit code has not
   always been reliable.** Both the Satellite and the Capsule certificates are
   validated on the Satellite Server, which is why the Capsule path in this role
   delegates those tasks. Older builds of the script had bugs where a failed
   check did not propagate a non-zero exit code, so the role treats both the
   return code and known failure markers in the output as failure.

---

## What it does

| Phase | Behavior |
| --- | --- |
| Preflight | Validates inputs; checks certificate expiry; warns when CN does not match the server FQDN; compares the new certificate against what the host serves today (skip-if-current, CA-change refusal); refuses to start when another `satellite-installer` or `satellite-maintain` is running. |
| Validate | Stages the material and runs `katello-certs-check -t satellite` or `-t capsule` on the Satellite Server. Stops here in validate-only mode. |
| Backup | `satellite-maintain service status` for the record, plus a tar of the certificate directories. |
| Deploy (Satellite) | `satellite-installer --scenario satellite --certs-server-cert/-key/-ca-cert --certs-update-server --certs-update-server-ca`. |
| Deploy (Capsule) | `capsule-certs-generate` on the Satellite, archive relayed to the Capsule, then `satellite-installer --scenario capsule --certs-tar-file --certs-update-server --foreman-proxy-foreman-base-url`. |
| Verify | Retrieves the certificate the endpoint now serves and asserts the serial matches what was deployed; checks the endpoint answers over TLS; writes a Markdown report. |

## Requirements

- ansible-core 2.16 or later.
- Collections: `community.crypto` (certificate inspection and verification),
  `community.general` (backup archives). Confirm they are in your execution
  environment; `ee-supported` does not necessarily carry both.
- Root (or `become`) on the Satellite and every Capsule, and SSH from the
  control node to all of them.
- The Satellite Server must be in the same inventory as the Capsules - Capsule
  tasks delegate to it.

## Inventory shape

```ini
[satellite]
satellite.example.com

[capsules]
capsule1.example.com
capsule2.example.com
```

```yaml
# group_vars/satellite.yml
satellite_cert_rotate_component: satellite

# group_vars/capsules.yml
satellite_cert_rotate_component: capsule
satellite_cert_rotate_satellite_host: satellite.example.com
satellite_cert_rotate_foreman_base_url: https://satellite.example.com
```

Each server needs its **own** certificate whose CN matches its own FQDN, so set
`satellite_cert_rotate_cert` / `_key` / `_ca_bundle` per host (host_vars) unless
you are using one SAN certificate that covers every FQDN.

## Examples

Validate only - no changes, no restart:

```
ansible-playbook -i inventories/satellite rotate_satellite_certs.yml \
  -e satellite_cert_rotate_validate_only=true
```

Rotate the Satellite, then the Capsules, with certificates from the same CA:

```
ansible-playbook -i inventories/satellite rotate_satellite_certs.yml \
  -e satellite_cert_rotate_accept_downtime=true
```

Rotate a single Capsule:

```
ansible-playbook -i inventories/satellite rotate_satellite_certs.yml \
  --limit capsule1.example.com \
  -e satellite_cert_rotate_accept_downtime=true
```

## Running it from automation controller

See `files/controller_job_template.yml` for a job template and survey using the
`ansible.controller` collection, and `files/satellite_cert_rotate_survey.json`
for the survey specification.

Attach a machine credential that can `become` root on the Satellite and the
Capsules. Certificate material belongs in a vaulted file in the project or in a
credential - not in a plaintext extra variable, and not in a survey answer.

If automation controller uses this Satellite as an inventory source or trusts
its CA, plan that rotation too: the controller's Satellite credential and any
custom CA in the execution environment or on the controller hosts will need the
new CA if the issuer changed.

## Where AAP changes the picture here, and where it does not

The rotation itself is three commands per server. `ansible-core` from a laptop
does that perfectly well, and for a single Satellite with no Capsules there is
little reason to reach for anything more.

What the platform adds is everything around the rotation: a survey so a platform
operator can run it without root shells or the private key on their laptop, RBAC
on who may run it, the certificate material coming from a credential, the job
output as the change record, a workflow that puts an approval node before the
Capsule stage, and - the part that actually matters at scale - the same platform
running the fleet-wide CA distribution to registered hosts that has to happen
before the CA changes. That last piece is what a point tool or a shell script
does not give you.

## License

MIT. Provided as-is, with no warranty and no support of any kind, from Red Hat
or from the author. Use at your own risk.
