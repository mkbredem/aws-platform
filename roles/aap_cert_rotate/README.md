# aap_cert_rotate

Rotate or renew the TLS certificates used by Red Hat Ansible Automation
Platform on container-based, operator-based (OpenShift Container Platform), and
RPM-based installations.

---

## NOT SUPPORTED - BY ANYONE

**This exists to illustrate what automating this procedure could look like.
It is not a product, not a deliverable, and not supported by anyone.**

- Not a Red Hat product. Not supported by Red Hat, Red Hat Support, or any Red
  Hat employee in any capacity - the author of this role included.
- No warranty, no service-level commitment, no maintenance, no bug fixes, and
  no commitment to answer questions about it, from Red Hat or from the author.
- **Not** validated against every Ansible Automation Platform version,
  topology, or configuration.
- Nothing here modifies Red Hat's support policy for an Ansible Automation
  Platform deployment, and running it does not entitle anyone to support for
  its behavior or for a platform it has touched.
- Anyone who runs it does so entirely at their own risk and is responsible for
  reviewing it, testing it in a lab first, and deciding whether it fits their
  environment.

It wraps the procedures published in *Operating Ansible Automation Platform* →
*Renewing and changing the SSL/TLS certificates*. Where this role and the
product documentation disagree, **the documentation is correct**. Every
inventory variable name and file path needs checking against the documentation
for the exact release in use - variable names and paths have changed between
releases.

There should be a restore path for the platform, not just for the certificates,
before anyone runs this.

---

## Caveats you need to read before running this

**You will take an outage.** For container-based and RPM-based installations
this role runs the installation program, which restarts every platform service.
Plan on a change window: installer runs commonly take 15-45 minutes depending
on topology, and the platform is unavailable for most of it. Running jobs are
interrupted. On operator-based installations the outage is shorter - the
operator reconciles the custom resource and restarts the affected pods - but it
is still an interruption for automation controller or automation hub.

**Do not run this from the platform it is rotating.** The installation program
restarts automation controller and platform gateway mid-run, so a job launched
from the same cluster kills itself partway through and you lose the job output
along with any chance of a clean rollback. Run it from:

- a workstation with `ansible-navigator` or `ansible-playbook`, or
- a separate Ansible Automation Platform cluster (a management or
  "platform-of-platforms" controller), or
- an execution node in a *different* cluster than the target.

The role enforces this: if `awx_job_id` is defined (the play is running inside
an automation controller job) it fails preflight unless you set
`aap_cert_rotate_separate_control_plane=true` to confirm you are deliberately
driving a different cluster.

**Nothing destructive happens until you acknowledge the downtime.** Preflight
fails unless `aap_cert_rotate_accept_downtime=true`. Set
`aap_cert_rotate_run_installer=false` for a staging-only dry run: the role
places the certificate material and edits the inventory file, and stops before
the installation program.

**The inventory file is edited in place.** The role uses
`community.general.ini_file` to set the TLS variables in the `[all:vars]`
section, taking a timestamped backup first. If your inventory file is generated
or under version control, either edit it there instead and run with
`aap_cert_rotate_mode` staging disabled, or diff the result before the
installation program runs.

**Certificates in front of a load balancer.** If TLS is terminated on a load
balancer rather than on the platform hosts, rotating platform certificates does
not change what users see. Verification will report the load balancer's
certificate and the serial-number assertion will fail. That is a correct
result, not a bug.

**Operator-based installations cover automation controller and automation hub
only.** Those are the components with a documented `route_tls_secret`
parameter. Platform gateway and Event-Driven Ansible are not handled by the
operator path in this role.

**PEM ordering matters.** Each certificate file must be the server certificate
first, followed by any intermediate CA certificates. The root CA goes in
`aap_cert_rotate_ca_cert` separately.

---

## What it does

| Phase | Behavior |
| --- | --- |
| Preflight | Validates install type, mode, component list, downtime acknowledgement, self-rotation guard, presence of certificate material and the installation program directory. |
| Input validation | Checks each certificate for expiry and minimum remaining validity, confirms the private key matches the certificate, and warns when the endpoint hostname is missing from the SAN list. |
| Backup | Timestamped copy of the inventory file and of every certificate and key currently on disk. |
| Rotation | Stages new material and updates the inventory, or moves existing certificates aside for regeneration, then runs the installation program. Operator path creates a TLS secret and patches the custom resource. |
| Verification | Retrieves the certificate each endpoint now serves, asserts it is not expired and matches the material you supplied, and confirms the endpoint answers over TLS. |
| Report | Writes a Markdown rotation report with the rollback path. |

## Requirements

- ansible-core 2.16 or later on the control node or execution environment.
- Collections: `community.general` (inventory edits), `community.crypto`
  (certificate validation and verification), `kubernetes.core` (operator path
  only).
- The default `ee-supported` execution environment does not necessarily carry
  all three. Confirm before running from automation controller, or build an
  execution environment that includes them. `ansible-navigator` on a
  workstation with the community collections installed is the simplest path.
- For container-based and RPM-based installations: SSH access to the host
  holding the installation program, as the user that performed the
  installation, and the original inventory file.
- For operator-based installations: a kubeconfig or an OpenShift API bearer
  token with permission to create secrets and patch the
  `AnsibleAutomationPlatform` resource.

## Key variables

See `defaults/main.yml` for the annotated full list.

| Variable | Default | Purpose |
| --- | --- | --- |
| `aap_cert_rotate_install_type` | `containerized` | `containerized`, `operator`, or `rpm`. |
| `aap_cert_rotate_mode` | `custom` | `custom` (you supply PEM material) or `regenerate` (installation program regenerates self-signed certificates). |
| `aap_cert_rotate_components` | gateway, controller, hub, eda | Components in scope. |
| `aap_cert_rotate_accept_downtime` | `false` | Must be true before the installation program runs. |
| `aap_cert_rotate_run_installer` | `true` | False stages changes without running the installation program. |
| `aap_cert_rotate_separate_control_plane` | `false` | Confirms the job is not rotating the cluster running it. |
| `aap_cert_rotate_cert_src` | `{}` | Per-component `cert` and `key` paths on the control node. |
| `aap_cert_rotate_wildcard_cert` / `_key` | `""` | Material used for any component without its own entry. |
| `aap_cert_rotate_ca_cert` | `""` | Root CA added to the platform trust store. |
| `aap_cert_rotate_installer_dir` | `~/aap-installer` | Installation program directory on the target host. |
| `aap_cert_rotate_verify_endpoints` | `{}` | `component: host:port` map checked after rotation. |

## Examples

Container-based, one wildcard certificate for the whole platform, from a
workstation:

```yaml
ansible-playbook -i inventories/aap rotate_aap_certs.yml \
  -e aap_cert_rotate_install_type=containerized \
  -e aap_cert_rotate_wildcard_cert=files/tls/aap-wildcard.pem \
  -e aap_cert_rotate_wildcard_key=files/tls/aap-wildcard.key \
  -e aap_cert_rotate_ca_cert=files/tls/corp-root-ca.pem \
  -e aap_cert_rotate_accept_downtime=true
```

Container-based, regenerate the installation program's self-signed
certificates:

```yaml
ansible-playbook -i inventories/aap rotate_aap_certs.yml \
  -e aap_cert_rotate_mode=regenerate \
  -e aap_cert_rotate_accept_downtime=true
```

RPM-based, dry run that stages material and edits the inventory only:

```yaml
ansible-playbook -i inventories/aap rotate_aap_certs.yml \
  -e aap_cert_rotate_install_type=rpm \
  -e aap_cert_rotate_run_installer=false
```

Operator-based on OpenShift, driven from localhost:

```yaml
ansible-playbook rotate_aap_certs.yml \
  -e aap_cert_rotate_install_type=operator \
  -e aap_cert_rotate_target_group=localhost \
  -e aap_cert_rotate_k8s_aap_name=myaap \
  -e aap_cert_rotate_k8s_namespace=ansible-automation-platform \
  -e aap_cert_rotate_k8s_kubeconfig=~/.kube/config \
  -e aap_cert_rotate_accept_downtime=true
```

## Running it from automation controller

Launch it from a **management** cluster, not the cluster being rotated. See
`files/controller_job_template.yml` for a job template and survey definition
using the `ansible.controller` collection, and
`files/aap_cert_rotate_survey.json` for the survey specification on its own.

Credentials to attach:

- a machine credential for the installation program host (container-based and
  RPM-based), and
- an OpenShift or Kubernetes API bearer token credential (operator-based).

Certificate material should come from a vaulted file in the project or from a
credential lookup - never from a plaintext extra variable.

## Rollback

Container-based and RPM-based: the previous inventory file and certificates are
in `{{ aap_cert_rotate_backup_dir }}/<timestamp>` on the target host. Restore
them and run the installation program again. That is a second outage.

Operator-based: the previous TLS secret is still in the namespace. Patch
`route_tls_secret` back to it and let the operator reconcile.

## Where AAP changes the picture here, and where it does not

The rotation itself is an installer run - `ansible-core` alone does that fine.
What the platform adds is the operational envelope: the survey turns the
change into a request a platform operator can submit without shell access to
the installer host, RBAC controls who may submit it, the certificate material
comes from a credential rather than someone's home directory, the job output is
the change record, and a workflow can chain the rotation to a notification and
an approval node. For one lab cluster rotated once a year, a workstation and a
playbook are enough; what the platform adds there is operational, not
technical.

## License

MIT. Provided as-is, with no warranty and no support of any kind, from Red Hat
or from the author. Use at your own risk.
