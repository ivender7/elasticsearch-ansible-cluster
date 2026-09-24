# Elasticsearch Cluster via Ansible

Test task: deploy a 3-node Elasticsearch cluster with Ansible so it comes up
automatically.

## What's here

- 3-node cluster (with 3 nodes you get a 2/3 quorum, so losing one node doesn't cause split-brain)
- Ubuntu 24.04 LTS as the base OS
- Elasticsearch 9.5.4
- TLS on both transport and HTTP layers, own CA generated via
  `elasticsearch-certutil`
- `elastic` user password managed through `ansible-vault`
- OS hardening: sysctl, ulimits, swap disabled, SSH hardened, ufw, unattended
  upgrades

## Running it

Lab environment is Vagrant + VirtualBox (3 VMs on a private network) :

```bash
vagrant up
ansible-galaxy collection install -r requirements.yml

echo 'your-vault-password' > .vault_pass && chmod 600 .vault_pass
ansible-vault encrypt inventories/dev/group_vars/all/vault.yml

ansible-playbook playbooks/site.yml
```

Check it worked:

```bash
curl --cacert .generated-certs/ca/ca.crt \
  -u elastic:<your-vault-password> \
  "https://192.168.56.11:9200/_cluster/health?pretty"
```

Should say `"status": "green"` and `"number_of_nodes": 3`.

## Layout

```
inventories/dev/      inventory + vars (vault.yml is encrypted)
roles/common/          OS-level stuff: sysctl, limits, swap, ssh, ufw, updates
roles/elasticsearch/   install, certs, config, systemd, password setup
playbooks/site.yml     entry point
.generated-certs/      CA + node certs, generated locally on each run (gitignored)
```

## A few things worth explaining

**Why disable ES's own auto-config.** Elasticsearch 8+/9+ auto-generates its
own CA and certs the moment the package is installed — you can actually see
this happen (`http.p12`, `http_ca.crt`, `transport.p12` show up in
`/etc/elasticsearch/certs` right after `apt install`, before Ansible even
touches TLS). Since I wanted to manage certs myself, `elasticsearch.yml`
already has explicit `xpack.security.*` settings by the time the service
first starts, so ES's own settings take over instead. The leftover
autoconfig files aren't used for anything — the role just deletes them so
they don't sit around and confuse anyone poking at the node later.

**Certs.** Generated once with `elasticsearch-certutil` on the first node,
then fetched to the controller and pushed out to every node — one cert per
node with the right SAN entries (hostname + IP), plus the CA public cert so
nodes can verify each other. The CA private key stays only on the
controller, in `.generated-certs/`, never touches the actual
nodes.

**Password.** First run: no password works yet, so the role resets `elastic`
to a random one and immediately changes it to whatever's in vault. Every run
after that, the vault password already works, so this whole block is
skipped — checked this by running the playbook twice in a row and confirming
`changed=0` on the second pass.

**Swap.** Turned off completely instead of just relying on
`bootstrap.memory_lock`. JVM garbage collection pauses are bad enough without
the OS deciding to page memory to disk mid-GC. Also set `LimitMEMLOCK` in a
systemd drop-in, since `/etc/security/limits.d` limits don't actually apply
to systemd-managed services — that one cost me some time to figure out.

**Firewall.** Default deny incoming. SSH open (would restrict to a
bastion/VPN in a real environment); 9200/9300 only reachable from the
cluster's own subnet.
