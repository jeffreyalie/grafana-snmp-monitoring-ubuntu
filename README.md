# grafana-snmp-ubuntu

Monitors Ubuntu hosts (CPU, RAM, swap, storage, NVIDIA GPU) over **SNMPv3** and shows everything as
`bargauge` panels in the **existing kube-prometheus-stack** Grafana, plus a separate alert-status panel
(solid green bar per host, solid red as soon as a *critical* alert fires).

Helm release name: `grafana-snmp-ubuntu`, namespace `monitoring`.

```
Ubuntu host (snmpd, SNMPv3)  <--udp/161--  snmp_exporter pod  <--/snmp?target=--  Prometheus (ScrapeConfig)
                                                                                      │  rules -> Alertmanager
                                                                                      ▼
                                                                         Grafana (dashboard ConfigMap, sidecar)
```

What the chart contains: `snmp_exporter` Deployment/Service/ConfigMap, a `ScrapeConfig`, a `PrometheusRule`,
and the dashboard ConfigMap (`grafana_dashboard: "1"`). It does **not** deploy Prometheus, Alertmanager or Grafana.

## Layout

```
hosts.yaml                 # targets (name, IP, gpu true/false) + scrapeSource; used by Ansible AND Helm
ansible/                   # role `snmpd`: install snmpd, SNMPv3 user, NVIDIA extends, ufw rule, self-test
helm-chart/                # chart (values.yaml holds thresholds, storage mount regex, alert rules)
.gitea/workflows/deploy.yaml
.gitea/workflows/destroy.yaml
```

## One-time setup

**1. OpenBao: SNMP credentials** (alphanumeric only, at least 8 characters; avoid `"` and `$`):

```bash
bao kv put homelab/snmp \
  username=prom \
  auth_password="$(openssl rand -hex 16)" \
  priv_password="$(openssl rand -hex 16)"
```

**2. OpenBao policy.** `policy write` overwrites, so list every existing path plus the new one:

```bash
bao policy write gitea-runner - <<'EOP'
path "homelab/data/lxd"      { capabilities = ["read"] }
path "homelab/data/seaweed"  { capabilities = ["read"] }
path "homelab/data/ansible"  { capabilities = ["read"] }
path "homelab/data/microk8s" { capabilities = ["read"] }
path "homelab/data/grafana"  { capabilities = ["read"] }
path "homelab/data/snmp"     { capabilities = ["read"] }
EOP
```

The workflow also uses `homelab/data/ansible` key `ssh_private_key` (already there).

**3. Edit `hosts.yaml`**: your Ubuntu IPs, `gpu: true` where an NVIDIA driver is installed.
Ansible connects as user `ansible` over SSH and needs passwordless sudo on those hosts.

**4. Gitea:** create the repo, push, and make sure the org secrets `VAULT_ADDR`, `VAULT_ROLE_ID`,
`VAULT_SECRET_ID` are available to it (they already are at org level).

**5. kube-prometheus-stack** must already be running in `monitoring` with the ScrapeConfig CRD
(`kubectl get crd scrapeconfigs.monitoring.coreos.com`). The chart labels its ScrapeConfig and
PrometheusRule `release: kube-prometheus-stack` (value `prometheusRelease`) so the default selectors pick them up.

## Deploy / destroy

* **Deploy:** push to `main` (touching `helm-chart/`, `ansible/` or `hosts.yaml`) or run
  *Actions → Deploy grafana-snmp-ubuntu*. The manual run has a `configure_hosts` checkbox to skip Ansible.
  Order: OpenBao login → Ansible on the hosts → Secret `snmp-exporter-auth` → `helm upgrade --install` → exporter restart.
* **Destroy:** *Actions → Destroy grafana-snmp-ubuntu*, type `destroy`. Removes the release (exporter, ScrapeConfig,
  rules, dashboard) and optionally the Secret. It does **not** uninstall snmpd from the hosts.

## Verify

```bash
microk8s kubectl get deploy,svc,scrapeconfig,prometheusrule -n monitoring | grep grafana-snmp-ubuntu
```

* Prometheus `http://<node>:30090/targets`: job `grafana-snmp-ubuntu` should be **UP** per host.
* PromQL: `ssCpuIdle`, `hrStorageUsed`, `nsExtendOutput1Line` should all return series.
* Grafana `http://<node>:30300`: dashboard **Ubuntu hosts (SNMP)**.
* Alertmanager `http://<node>:30093`: alerts appear here when they fire.

Test one exporter module by hand:

```bash
microk8s kubectl port-forward -n monitoring svc/grafana-snmp-ubuntu-snmp-exporter 9116:9116
curl 'localhost:9116/snmp?target=10.0.0.101&module=ubuntu&auth=ubuntu_v3' | grep -E '^(ssCpuIdle|nsExtend)'
```

## Alerts and the alert panel

* Every rule in `values.yaml` produces `<Name>Warning` and `<Name>Critical` (thresholds 80/90, GPU temp 75/85),
  plus `UbuntuHostDown` (critical, target unreachable for 2m).
* The **Alert status (critical)** panel shows one bar per host with value `100 + <number of firing critical alerts>`.
  OK = 100 (full green bar, text "OK"); any critical alert = above 100 (full red bar, text "CRITICAL").
  Warnings never turn it red.
* Alertmanager only *routes* notifications. The monitoring stack currently sends everything to the `null`
  receiver, so alerts show up in Alertmanager and on the panel but nothing is sent. Add a receiver in the
  `monitoring` repo's `values.yaml` and route on `severity: critical` if you want notifications.
* Changing thresholds: edit `values.yaml` for alerts and the `steps`/descriptions in
  `helm-chart/dashboards/ubuntu-snmp.json` for bar colours (they are separate).

## Notes and limits

* **GPU:** one value per host (the highest across GPUs). Needs `nvidia-smi` at `/usr/bin/nvidia-smi`.
* **Storage:** only mounts matching `storage.mountRegex` (default `/`, `/boot*`, `/home*`, `/mnt*`, `/srv*`, `/data*`).
  If you change it, change the regex in the dashboard JSON too.
* **RAM %** = `100 * (1 - (free + buffers + cached) / total)` from UCD-SNMP-MIB.
* **Credentials on the hosts:** Ansible puts the SNMP passphrases in `/etc/snmp/snmpd.conf` (root:Debian-snmp, 0640)
  as a `createUser` line. To rotate: change `homelab/data/snmp`, then on each host stop snmpd, delete the `usmUser`
  lines from `/var/lib/snmp/snmpd.conf`, and re-run the deploy workflow.
* TLS to OpenBao is skipped (`curl -k`) like your other workflows; SSH host keys are not checked in Ansible.

## Troubleshooting

| Symptom | Check |
|---|---|
| Target DOWN, exporter log `request timeout` | udp/161 blocked (`ufw`, `scrapeSource` IP), `agentaddress` still localhost, wrong IP in `hosts.yaml` |
| Exporter log `unknown user` / `wrong digest` | credentials in OpenBao differ from what snmpd localised; do the rotation steps above |
| Exporter pod CrashLoop, config error | `kubectl logs` on the pod; empty `snmp-exporter-auth` keys fail config validation |
| GPU panels empty | `snmpwalk ... 1.3.6.1.4.1.8072.1.3.2.3.1.1` on the host; `gpu: true` set? `nvidia-smi` works as user `Debian-snmp`? |
| Storage panel empty | `snmpwalk ... 1.3.6.1.2.1.25.2.3.1.3` to see the real `hrStorageDescr` names, adjust `storage.mountRegex` |
| ScrapeConfig/rules ignored | Prometheus selectors differ from `release: kube-prometheus-stack`; set `prometheusRelease` |
| Workflow: `ssh_private_key is empty or not a key` | fix `homelab/data/ansible` (value must be the private key text) |

---
## Ubuntu end SNP manual steps
These steps are already in GHA deploy workflow
Here is the Ubuntu side, since it doesn’t depend on how the chart ends up. The data path is Prometheus → snmp_exporter (a pod in the chart) → snmpd on each Ubuntu host. Grafana only talks to Prometheus.

1. Install the agent

```bash
sudo apt update
sudo apt install -y snmpd snmp
```
2. Create a read-only SNMPv3 user (stop the service first, since snmpd rewrites this file on start):

```bash
sudo systemctl stop snmpd
echo 'createUser prom SHA "CHANGE_AUTH_PASS" AES "CHANGE_PRIV_PASS"' | sudo tee -a /var/lib/snmp/snmpd.conf
```
Use passphrases of at least 8 characters. On start, snmpd replaces this line with a hashed version.

3. Edit /etc/snmp/snmpd.conf. Ubuntu’s default listens on localhost only and ships with community strings, so remove the rocommunity lines and set:

```conf
agentaddress udp:161
rouser prom priv .1
```
**Storage and CPU/RAM come from the built-in HOST-RESOURCES and UCD-SNMP MIBs.**
includeAllDisks 10%

4. GPU (NVIDIA example). snmpd has no GPU MIB, so expose nvidia-smi output with extend:

```conf
extend gpu-util /usr/bin/nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits
extend gpu-mem  /usr/bin/nvidia-smi --query-gpu=memory.used,memory.total --format=csv,noheader,nounits
extend gpu-temp /usr/bin/nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits
```

The values appear under NET-SNMP-EXTEND-MIB and snmp_exporter converts them to numbers with a regex. This works, but a plain node_exporter with a GPU exporter would be simpler if you ever want to drop SNMP for GPUs.

5. Start it and open the firewall to the MicroK8s node only. Pod traffic leaves with the node’s IP:

```bash
sudo systemctl enable --now snmpd
sudo ufw allow from 10.206.213.243 to any port 161 proto udp
```
6. Test from the MicroK8s node (apt install snmp there):

```bash
snmpwalk -v3 -l authPriv -u prom -a SHA -A 'CHANGE_AUTH_PASS' -x AES -X 'CHANGE_PRIV_PASS' <ubuntu-ip> 1.3.6.1.4.1.2021.10   # load average
snmpwalk -v3 -l authPriv -u prom -a SHA -A 'CHANGE_AUTH_PASS' -x AES -X 'CHANGE_PRIV_PASS' <ubuntu-ip> 1.3.6.1.4.1.8072.1.3.2   # GPU extends
```

Using your Ansible secrets: I’d turn steps 1 to 5 into a small Ansible role. The deploy workflow would pull homelab/data/ansible from OpenBao, run the role against the hosts, and store the SNMP credentials at a new path, homelab/data/snmp. Remember that bao policy write overwrites, so the gitea-runner policy needs all existing paths plus the new one.
