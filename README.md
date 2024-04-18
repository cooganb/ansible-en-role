# ansible-en-role

Ansible role to deploy and configure zkSync Era External Node, including DB instance setup on the same machine, Traefik as reverse proxy, and Prometheus monitoring (PostgreSQL exporter, cAdvisor, Traefik, External Node native metrics, and VictoriaMetrics vmagent to scrape all of them).

Make sure to configure Prometheus remote write endpoint to send metrics to centralized metrics storage.

Role has been tested and used internally on bare metal Hetzner instances.

## Requirements

This role has been tested on:

* Ubuntu 22.04, Jammy Jellyfish; Ansible 2.13.9

## Usage

For a very simple minimal working example, see example_playbooks directory

Minimal required variables that have to be set:

```yaml
database_name: ""
database_username: ""
database_password: ""
eth_l1_url: ""
main_node_url: ""
l1_chain_id: ""
l2_chain_id: ""
```

Additional arbitrary environment variables can be passed to External Node container:

```yaml
additional_env_vars:
  - { name: "EN_ADDITIONAL_VAR1", value: "Value1" }
  - { name: "EN_ADDITIONAL_VAR2", value: "Value2" }
  - { name: "EN_ADDITIONAL_VAR3", value: "Value3" }
```

Please refer to [External Node docs](https://github.com/matter-labs/zksync-era/tree/main/docs/guides/external-node/prepared_configs) to find values for different zkSync Era chains.

If you want to use monitoring (which we highly recommend), you have to change these variables:

```yaml
# Monitoring options section
enable_monitoring: true
node_name: "some-unique-node-identifier"
prometheus_remote_write: true
prometheus_remote_write_url: "https://metrics.example.org"
prometheus_remote_write_auth: true
prometheus_remote_write_auth_username: "admin"
prometheus_remote_write_auth_password: "password"
prometheus_remote_write_common_label: "matterlabs"
```

This role also has the option to secure your server and allow traffic only from specified IP address in case if you want
to use some load balancer in front of your node, while not having fancy cloud security groups at your disposal:

```yaml
# Security options
use_predefined_iptables: true
disable_ssh_password_auth: true
iptables_packages:
  - iptables
  - iptables-persistent
# Variable to be used to accept external traffic only from single specified IP
loadbalancer_ip: "1.2.3.4"
```

In most cases, you'd want to change PostgreSQL parameters, so you can do it using `postgres_arguments` variable, eg:

```yaml
postgres_arguments:
  - log_error_verbosity=terse
  - -c
  - max_connections=256
  - -c
  - shared_buffers=47616MB
  - -c
```

We recommend using pgtune [online](https://pgtune.leopard.in.ua/) or [self-hosted](https://github.com/le0pard/pgtune) version with "Online transaction processing system" preset as a good starting point for generating optimal config for your hardware.

If you want to use basic auth for inbound requests, you have to change next variables:

```yaml
enable_basic_auth: true
basic_auth_secret: "htpasswd-generated-secret"
```

Basic auth secret can be generated by `htpasswd` and `sed` for interpolation:
```echo $(htpasswd -nb <username> <password>) | sed -e s/\\$/\\$\\$/g```

## Step-by-step guide

1. Install the ansible collection on your machine from where you will run ansible:
`ansible-galaxy collection install community.general`

2. Prepare the latest database backup on your host. you can download it from our public GCS buckets:
Skip this step if you are recovering from a snapshot!

* [Era Mainnet latest dump](https://storage.googleapis.com/zksync-era-mainnet-external-node-backups/external_node_latest.pgdump)
* [Era Sepolia Testnet latest dump](https://storage.googleapis.com/zksync-era-boojnet-external-node-snapshots/external_node_latest.pgdump)

Downloaded dump file should be placed into `{{ storage_directory }}/pg_backups` directory (`/usr/src/en/pg_backups` by default)

3. **OPTIONAL**: If you already have running node, you can copy its tree and state directory to a new host's `{{ storage_directory }}/db`. (`/usr/src/en/db` by default)
Skip this step if you are recovering from a snapshot!

**Keep in mind that tree and state should be older than PostgreSQL database backup.**

4. Run ansible-playbook using this role. We recommend encrypting next variables with ansible-vault or some another way:

```yaml
database_username
database_password
eth_l1_url
vm_auth_username
vm_auth_password
```

5. Connect to your host, and see status of `postgres` container. It can take a lot of time before PostgreSQL database backup will be restored (hours to days, depending on your disk throughput and IOPS), after which PostgreSQL server will be ready for use. Once `postgres` becomes "healthy", `external_node` runs automatically.

## Snapshots Recovery

Example config enabling recovery from a snapshot:

```yaml
- enable_snapshots_recovery: true
- snapshots_bucket_base_url: "snapshots-bucket-name"
```

Snapshot buckets:

* Era Mainnet: `zksync-era-mainnet-external-node-snapshots`
* Era Sepolia Testnet: `zksync-era-boojnet-external-node-snapshots`

## Example Playbook

```yaml
---
- hosts: all
  become: true
  vars:
    loadbalancer_ip: "1.2.3.4"
    use_predefined_iptables: true
    enable_monitoring: false
    database_name: "mainnet2"
    main_node_url: "https://zksync2-mainnet.zksync.io"
    l2_chain_id: "324"
    l1_chain_id: "1"
    enable_tls: false
  vars_files:
    - secrets/mainnet_secrets.yml
  roles:
    - external_node
```

## License

Ansible role for zkSync Era External Node is distributed under the terms of either

* Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
* MIT license ([LICENSE-MIT](LICENSE-MIT) or <https://opensource.org/blog/license/mit/>)

at your option.
