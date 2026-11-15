# Command cheat sheet

The commands that come up again and again when running the fleet, each with a one-line note and a link to the page that explains it. Use it to find a command you have run before. When you're making a change for the first time, follow the linked page instead.

## Placeholders {#placeholders}

| Placeholder | Value |
| --- | --- |
| `<host>` | The host's name in the inventory, such as `km01` |
| `<address>` | The host's address, such as `172.16.7.101` |
| `<target>` | A host, a group, or a comma-separated list of either |
| `<tags>` | One or more `site.yml` stages: `vms`, `wait`, `nixos`, `komodo` |
| `<admin>` | The admin account, `abbr_name` followed by `admin` |
| `<vmid>` | The VM's ID in Proxmox |
| `<generation>` | A generation number from `nixos-rebuild list-generations` |
| `<stack>` | A stack's project name, from `docker compose ls` |
| `<service>` | A service in that stack, from `docker compose -p <stack> ps -a` |
| `<secret-name>` | A key in the secrets file, such as `komodo-onboarding-key` |
| `<value>` | The value to store under that key |
| `<message>` | A commit message that says what changed |

## Before you start {#setup}

Open the tools shell, load the environment file, and work from the fleet-ansible checkout. Every command on the control node below runs from there unless its section says otherwise. See [The control shell](../fleet-bootstrap/foundation/control-shell.md#tools).

```bash
nix shell nixpkgs#git nixpkgs#ansible nixpkgs#opentofu nixpkgs#sops nixpkgs#age \
  nixpkgs#sshpass nixpkgs#mkpasswd
source ~/.config/fleet/env
cd ~/src/fleet-ansible
```

Pull all three checkouts before a run, so it uses what is on GitHub:

```bash
for repo in fleet-ansible fleet-nixos fleet-private; do git -C ~/src/$repo pull; done
```

## Running hosts {#run}

Build or update a host from start to finish: its VM, its NixOS, and its stacks. See [Running part of a run](ansible/run-part-of-a-run.md).

```bash
ansible-playbook -i ../fleet-private/hosts.yml site.yml -e target=<host>
```

Preview a run without changing anything. Check mode stops at OpenTofu's plan.

```bash
ansible-playbook -i ../fleet-private/hosts.yml site.yml -e target=<target> --check
```

Run only some stages, such as `--tags nixos` to deploy NixOS or `--tags komodo` to deploy the stacks:

```bash
ansible-playbook -i ../fleet-private/hosts.yml site.yml -e target=<target> --tags <tags>
```

Add `-v` to any run to see more of what a failing task did. See [Reading a failed run](ansible/read-a-failed-run.md).

Show every value a host gets from the inventory:

```bash
ansible-inventory -i ../fleet-private/hosts.yml --host <host>
```

## Writing the generated files {#sync}

After a change to the inventory, write the fleet's NixOS file and Komodo files again, then review and commit them. See [The two sync playbooks](ansible/index.md#sync-playbooks).

```bash
ansible-playbook -i ../fleet-private/hosts.yml nixos-sync.yml
ansible-playbook -i ../fleet-private/hosts.yml komodo-sync.yml
git -C ../fleet-private add hosts.yml group_vars/ nixos/ komodo/
git -C ../fleet-private diff --cached
git -C ../fleet-private commit -m "<message>"
git -C ../fleet-private push
```

## The installer ISO {#iso}

Build the installer ISO and copy it to every Proxmox host. Run it again after a change to `ansible_ssh_public_keys` or `admin_ssh_public_keys`, once `nixos-sync.yml` has written them. See [Rebuild the ISO](../fleet-bootstrap/foundation/proxmox-and-installer.md#rebuild-iso).

```bash
ansible-playbook -i ../fleet-private/hosts.yml provision.yml \
  -e target=pve_host --tags pve-installer-iso,users -u root --ask-pass
```

Force a new build when nothing has changed, by deleting the ISO from the nix store, then run the command above again:

```bash
iso=$(nix run ../fleet-nixos#build-installer -- --fleet ../fleet-private)
nix store delete "${iso%/iso/*}"
```

## NixOS from the control node {#nixos}

Deploy a host's configuration without running `site.yml`. Add `--action test` to try it until the next restart, or `--action boot` to apply it at the next restart. See [Deploy to a host](../fleet-bootstrap/procedures/update-the-fleet.md#deploy).

```bash
nix run ../fleet-nixos#deploy-host -- --fleet ../fleet-private <host> <address>
```

Ask whether an address runs the installer, an installed host, or nothing:

```bash
nix run ../fleet-nixos#host-state -- <address>
```

Make the SSH host keys of a new host. See [The host's SSH keys](../fleet-bootstrap/concepts/secrets-with-sops.md#host-keys).

```bash
nix run ../fleet-nixos#new-host-key -- --fleet ../fleet-private <host>
```

Send a built host back to the installer, so the next run installs it again. It wipes the OS disk. See [Rebuilding a VM](../fleet-bootstrap/procedures/rebuild-a-vm.md#reset).

```bash
nix run ../fleet-nixos#reset-host -- --yes-wipe <host> <address>
```

## In the fleet-nixos checkout {#flake}

These run from `~/src/fleet-nixos`. See [Finding your way around](nixos/index.md#around-control).

Check that every host's configuration still evaluates against your private repo:

```bash
nix flake check --no-build --no-write-lock-file \
  --override-input fleet git+file://$HOME/src/fleet-private
```

Read one value of a host's configuration:

```bash
nix eval .#nixosConfigurations.<host>.config.fleet.features \
  --override-input fleet git+file://$HOME/src/fleet-private \
  --no-write-lock-file
```

Move the flake's inputs to their newest versions, then check and commit the lock file. See [Move the pin](../fleet-bootstrap/procedures/update-the-fleet.md#lock).

```bash
nix flake update
git add flake.lock
git commit -m "Update the flake's inputs"
git push
```

## Secrets {#secrets}

These run from `~/src/fleet-private`, where sops finds `.sops.yaml`. See [Changing a secret](../fleet-bootstrap/concepts/secrets-with-sops.md#edit).

Edit the secrets every host reads, or the ones the Proxmox run reads:

```bash
sops secrets/fleet.yaml
sops group_vars/all/secrets.sops.yaml
```

Set one value without an editor, or read one back:

```bash
sops set secrets/fleet.yaml '["<secret-name>"]' '"<value>"'
sops decrypt --extract '["<secret-name>"]' secrets/fleet.yaml
```

Hash a password for `server-password-hash` or `client-password-hash`:

```bash
nix shell nixpkgs#mkpasswd -c mkpasswd -m yescrypt
```

## On a NixOS host {#on-a-host}

These run on the host, after `ssh <admin>@<address>`. See [On a host](nixos/index.md#around-host).

Check what is running and whether anything failed:

```bash
nixos-rebuild list-generations
systemctl --failed
docker compose ls
docker ps
```

Go back to the previous generation. See [Rolling back](../fleet-bootstrap/procedures/update-the-fleet.md#rollback-switch).

```bash
sudo nixos-rebuild switch --rollback --no-reexec
```

Go to a given generation instead:

```bash
sudo nix-env --profile /nix/var/nix/profiles/system --switch-generation <generation>
sudo /nix/var/nix/profiles/system/bin/switch-to-configuration switch
```

Tell whether the running kernel differs from the deployed one, which means the host needs a restart. See [Restart the host](../fleet-bootstrap/procedures/update-the-fleet.md#reboot).

```bash
readlink /run/booted-system/kernel /run/current-system/kernel
```

Read Komodo Periphery's log, and list the firewall's rules:

```bash
sudo journalctl -u komodo-periphery -n 50
sudo iptables -S nixos-fw
```

Look at one stack's containers and logs. See [Looking inside a stack on a host](docker-compose/inspect-a-stack.md).

```bash
docker compose -p <stack> ps -a
docker compose -p <stack> logs --tail 50 <service>
```

## On a Proxmox host {#proxmox}

These run on the Proxmox host as root.

Show a VM's state and settings, open its serial console, or restart it hard. Leave the console with `Ctrl+O`.

```bash
qm status <vmid>
qm config <vmid>
qm terminal <vmid>
qm reset <vmid>
```

List the ISOs on the host:

```bash
pvesm list local --content iso
```

## OpenTofu by hand {#opentofu}

Make a checkout to read the VMs' state or plan one VM. See [Reading a plan before applying](opentofu/read-a-plan.md#checkout).

```bash
rm -rf /tmp/ansible-fleet-opentofu-checkout
git clone --depth 1 https://github.com/myah-mitchell/fleet-opentofu \
  /tmp/ansible-fleet-opentofu-checkout
cd /tmp/ansible-fleet-opentofu-checkout/envs/prod
cp ~/src/fleet-private/opentofu/prod.tfvars private.auto.tfvars
tofu init
tofu state list
tofu plan -target='module.vm["<host>"]'
```
