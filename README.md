# blog-lvm-expand

Companion code for the blog post
[Expanding Disk Space on LVM: A Step-by-Step Guide](https://blog.bsdserver.nl/expanding-disk-space-on-lvm-a-step-by-step-guide/).

The `lvm_expand` role does the six steps from the post on a running system:

1. check that the new disk is empty (`lsblk`), and optionally clear old partition tables with
   `wipefs --all --backup` (off unless you set `lvm_expand_wipe: true`);
2. and 4. initialise it as a physical volume and add it to the volume group (`pvcreate`, `vgextend`);
5. and 6. grow the logical volume and its filesystem (`lvextend` plus `resize2fs`).

Terraform creates an Ubuntu 24.04 VM on Proxmox VE with two empty data disks. The `lvm_demo` role turns
the first one into a volume group with a mounted ext4 volume, so there is something to extend; the
second disk is the new disk from the post.

## Requirements

- Terraform >= 1.9 and a Proxmox VE API token (the bpg/proxmox provider, pinned in `terraform/versions.tf`).
- Ansible (ansible-core 2.17 or later; tested with 2.22) with the collections in `ansible/requirements.yml`.
- A Debian or Ubuntu host (tested on Ubuntu 24.04).

## Quick start

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # endpoint, token, node, IP, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

cd ../ansible
ansible-galaxy collection install -r requirements.yml -p collections
ansible-playbook site.yml
ansible-playbook verify.yml    # pvscan, vgdisplay and df from the post, as assertions
```

On your own machine: set `lvm_demo_enabled: false` in `group_vars/all/vars.yml` and set
`lvm_expand_vg`, `lvm_expand_disks`, `lvm_expand_lv` and `lvm_expand_lv_size` (for example
`ubuntu-vg`, `[/dev/sdc]`, `lv_var` and `40g`, as in the post).

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `lvm_expand_vg` | `ubuntu-vg` | Volume group to extend |
| `lvm_expand_disks` | `[/dev/sdc]` | New, empty disks to add |
| `lvm_expand_wipe` | `false` | Remove old signatures and partition tables from those disks first |
| `lvm_expand_lv` | `lv_var` | Logical volume to grow |
| `lvm_expand_lv_size` | `+100%FREE` | New size: an absolute size (`40g`) or `+100%FREE` |
| `lvm_demo_enabled` | `true` in `group_vars` | Build the practice VG `data-vg` on `/dev/sdb` first |

A relative size such as `+30G` (the `lvextend -L+30G` from the post) would grow the volume again on every
run. Use an absolute size or `+100%FREE` so that running the playbook again changes nothing.

The role refuses disks that are partitioned, carry a filesystem signature or belong to another volume group,
unless `lvm_expand_wipe` is set. `wipefs --backup` writes the removed signatures to `/root/wipefs-*.bak`.

## Tested

Lab-tested on Ubuntu 24.04 (LVM 2.03.16) with ansible-core 2.22 and community.general 14.0.0: the first run
builds the practice VG and extends it, the second run reports `changed=0`, and `verify.yml` passes. A
partitioned disk is refused without `lvm_expand_wipe` and wiped and added with it. Terraform is checked
with `fmt -check`, `validate` and `tflint`; `terraform apply` was not run as part of these tests.

## License

MIT
