# Lessons Learned: Selectel Cloud + Terraform GPU Provisioning

Hard-won knowledge from provisioning GPU servers on Selectel Cloud with Terraform. Reference this before creating or modifying Selectel infrastructure.

---

## Cloud-Init

### DNS is not ready when `packages:` runs

Cloud-init executes stages in order: `init` → `config` → `final`. The `packages:` directive runs during the `config` stage, **before** `runcmd`. On Selectel VMs (especially GPU flavors), the network/DNS may not be fully initialized when `config` runs. This causes all `apt-get install` calls to fail with `Could not resolve 'mirror.selectel.ru'`.

**Fix:** Do NOT use the `packages:` section. Move all package installs into `runcmd` after an explicit DNS readiness wait:

```yaml
package_update: false
package_upgrade: false

runcmd:
  - for i in $(seq 1 30); do nslookup mirror.selectel.ru >/dev/null 2>&1 && break; sleep 10; done
  - for i in $(seq 1 5); do apt-get update -qq 2>/dev/null && break; sleep 15; done
  - DEBIAN_FRONTEND=noninteractive apt-get install -y ca-certificates curl git docker.io docker-compose-v2
```

### Cloud-init takes 10+ minutes on GPU setups

NVIDIA driver install (DKMS kernel module build) + Docker image pull (`ostris/aitoolkit:latest` is ~8 GB) can take 10–15 minutes total. SSH wait loops must account for this — 30 retries × 10s (5 min) is not enough.

**Use** 60 retries × 10s (10 min) for SSH, plus `ServerAliveInterval=15` and `ServerAliveCountMax=40` to prevent SSH drops during the long cloud-init wait.

### Always add an error marker file

Cloud-init runcmd should write to `/root/cloud-init-error` on failure and `/root/cloud-init-ready` on success. The provisioning script can poll both to detect failures early instead of waiting for the full timeout.

---

## Terraform + Selectel

### Flavor names are not guessable

Selectel GPU flavor names follow no obvious convention. Examples:
- RTX 4090 (8 vCPU, 32 GB RAM): `GL10.8-32768-0-1GPU`
- RTX 6000 Ada (12 vCPU, 64 GB RAM): `GL14.12-65536-1GPU`

Note: the prefix (`GL10`, `GL14`) does NOT correspond to vCPU count or GPU model in any predictable way. The `-0-` in some names is absent in others.

**To discover flavors:** Create a project + service user, get a project-scoped Keystone token, then query:
```
GET https://ru-7.cloud.api.selcloud.ru/compute/v2.1/flavors/detail
```

An empty project (no network/VM) is enough — flavors are region-level. Destroy the project after querying.

### `data` source failures block `terraform destroy`

If `flavor_name` references a non-existent flavor, the `data.openstack_compute_flavor_v2` lookup fails — and it fails on **destroy** too, not just apply. This creates orphan resources that can't be removed via normal `terraform destroy`.

**Workaround in scripts:**
1. Remove OpenStack resources from state: `terraform state list | grep '^openstack_' | xargs -I{} terraform state rm {}`
2. Destroy remaining Selectel resources with an empty flavor: `terraform destroy -var 'flavor_name=' -auto-approve`

Both `provision.sh` and `destroy.sh` should implement this as automatic fallback.

### Orphan project cleanup

If a project gets created but terraform state is lost or corrupted, you can't re-create it (409 `already_exists`). To find and remove orphan projects:

1. Get an unscoped Keystone token (service user credentials)
2. List projects: `GET /identity/v3/auth/projects` (with the project-scoped token from step 1)
3. Import into a **minimal** Terraform config (Selectel provider only — no OpenStack provider, which would fail on computed `tenant_id`)
4. `terraform destroy`

The Keystone `DELETE /identity/v3/projects/{id}` endpoint returns 403 for service users — only the VPC resell API or Terraform can delete projects.

### OpenStack provider depends on Selectel resources

The `provider "openstack"` block references `selectel_vpc_project_v2.ai_toolkit.id` and `selectel_iam_serviceuser_v1.ai_toolkit.name`. This means:
- `terraform import` of individual resources fails if OpenStack provider can't authenticate
- Importing orphan projects requires a separate Terraform workspace with only the Selectel provider

### Static passwords in `~/.profile`

Never use command substitution for passwords:
```bash
# BAD — regenerates on every `source ~/.profile`
export PASSWORD="$(openssl rand -base64 24)#Ax1"

# GOOD — static, survives re-sourcing
export PASSWORD="UJXpJCEQcQeS3grwIQptSuCGwvPTPPFa#Ax1"
```

Dynamic passwords cause auth failures when the password changes between `terraform apply` and `terraform destroy`.

---

## Selectel API

### API endpoints by function

| Purpose | URL | Auth header |
|---------|-----|-------------|
| Keystone (auth, tokens, projects) | `https://cloud.api.selcloud.ru/identity/v3/` | `X-Auth-Token` |
| Compute (flavors, instances) | `https://{region}.cloud.api.selcloud.ru/compute/v2.1/` | `X-Auth-Token` (project-scoped) |
| VPC Resell (project CRUD) | `https://api.selectel.ru/vpc/resell/v2/` | `X-Token` (account token) |

### VPC API can be very slow or hang

The `api.selectel.ru` VPC/IAM endpoints can hang for 30+ seconds or time out entirely from some networks. Prefer Keystone endpoints (`cloud.api.selcloud.ru`) when possible.

### Token scoping matters

- **Unscoped tokens** (no `scope` in auth request): Cannot list projects via `/identity/v3/projects`.
- **Domain-scoped tokens**: Can list projects via `/identity/v3/auth/projects`.
- **Project-scoped tokens**: Required for compute API calls (flavors, instances).

The `/identity/v3/auth/projects` endpoint lists projects accessible to the authenticated user. The `/identity/v3/projects` endpoint requires admin privileges and returns 401 for service users.

---

## NVIDIA / CUDA

### CUDA OOM is about VRAM, not fragmentation

`torch.OutOfMemoryError` during VAE decode in flux/image models typically means the GPU physically doesn't have enough VRAM. The RTX 4090 (24 GB) ran out during QwenImage training. Moving to RTX 6000 Ada (48 GB) resolved it.

Adding `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` helps with fragmentation but cannot fix true OOM.

### NVIDIA driver packages

On Ubuntu 24.04, `nvidia-driver-590` installs driver 590.48.01 with CUDA 13.1. The package name format is `nvidia-driver-{major_version}`. Always install `linux-headers-$(uname -r)` alongside for DKMS module build.

After install: `dkms autoinstall`, `modprobe nvidia`, `modprobe nvidia_uvm` — then verify with `nvidia-smi`.

---

## Gitignore Patterns

### Terraform timestamped backup files

Terraform creates backups like `terraform.tfstate.1772092063.backup` (timestamp in the middle). The pattern `*.tfstate.backup` does NOT match these.

**Use:** `*.tfstate.*backup` to catch both `terraform.tfstate.backup` and `terraform.tfstate.{timestamp}.backup`.
