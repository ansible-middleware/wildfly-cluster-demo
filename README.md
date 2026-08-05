# WildFly Cluster Demo

[![CI](https://github.com/ansible-middleware/wildfly-cluster-demo/actions/workflows/ci.yml/badge.svg)](https://github.com/ansible-middleware/wildfly-cluster-demo/actions/workflows/ci.yml)
[![License: GPL-2.0](https://img.shields.io/badge/License-GPL--2.0-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)
[![WildFly Collection](https://img.shields.io/badge/Ansible_Collection-middleware__automation.wildfly-ee0000.svg)](https://galaxy.ansible.com/ui/repo/published/middleware_automation/wildfly/)
[![WildFly Version](https://img.shields.io/badge/WildFly-40.0.0.Final-orange.svg)](https://www.wildfly.org/)

---

A fully-working Ansible demo that deploys a **three-instance WildFly 40 HA cluster** (standalone-ha mode with JGroups multicast) on one or more target hosts.

The demo uses the official [middleware_automation.wildfly](https://github.com/ansible-middleware/wildfly) Ansible collection (v1.5.14+) and showcases:

* Automated download and installation of WildFly 40
* Three co-located systemd-managed instances with automatic port offsetting
* High-availability configuration via `standalone-ha.xml` and JGroups multicast
* Application deployment via the `wildfly_app_deploy` role (jboss-cli under the hood)
* Post-install validation using the WildFly management HTTP API
* Molecule-based integration tests running against a UBI 9 container

---

## Cluster Topology

```
           ┌──────────────────────────────────────────────────────┐
           │                   Target Host(s)                     │
           │                                                      │
           │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │
           │  │  wildfly-0  │ │  wildfly-1  │ │  wildfly-2  │     │
           │  │  HTTP :8080 │ │  HTTP :8180 │ │  HTTP :8280 │     │
           │  │  Mgmt :9990 │ │  Mgmt:10090 │ │  Mgmt:10190 │     │
           │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘     │
           │         └───────────────┼───────────────┘            │
           │               JGroups Multicast                      │
           │               230.0.0.4 (UDP)                        │
           └──────────────────────────────────────────────────────┘
```

Each instance runs as a separate systemd unit (`wildfly-0.service`, `wildfly-1.service`, `wildfly-2.service`) under the `wildfly` system user.

---

## Prerequisites

| Requirement | Version |
|:---|:---|
| Ansible | >= 2.16 |
| Python | >= 3.10 |
| Target OS | RHEL 8/9, Fedora 38+, or compatible UBI image |
| Java (on target) | Java 21 (installed automatically by the collection) |
| Internet access (controller) | To download WildFly archive from GitHub Releases |

---

## Quick Start

### 1. Install the Ansible collection

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure your inventory

Edit the `inventory` file to point to your target hosts:

```ini
[all]
my-host.example.com ansible_user=ec2-user ansible_become=true
```

For a local-only demo (single machine):

```ini
[all]
localhost ansible_connection=local
```

### 3. Prepare the helloworld WAR

The demo deploys a small `helloworld.war` to each instance. Molecule builds one automatically in the prepare step. For a manual run, build the [WildFly quickstart](https://github.com/wildfly/quickstart) from source (releases ship source zips only):

```bash
git clone --depth 1 --branch 40.0.0.Final https://github.com/wildfly/quickstart.git
cd quickstart/helloworld
mvn clean package
cp target/helloworld.war /tmp/
```

### 4. Run the playbook

```bash
ansible-playbook -i inventory playbook.yml
```

After a successful run you can access:
- Instance 0 – http://YOUR_HOST:8080/helloworld/
- Instance 1 – http://YOUR_HOST:8180/helloworld/
- Instance 2 – http://YOUR_HOST:8280/helloworld/

---

## Configuration

All tuneable variables live in [`vars/vars.yml`](vars/vars.yml). Key variables:

| Variable | Default | Description |
|:---|:---|:---|
| `wildfly_version` | `40.0.0.Final` | WildFly release to install |
| `wildfly_install_workdir` | `/opt/wildfly/` | Installation root on target host |
| `wildfly_user` / `wildfly_group` | `wildfly` / `wildfly` | POSIX user/group for the service |
| `wildfly_java_version` | `21` | JDK major version (installed via RPM) |
| `wildfly_java_opts` | `-Xmx1024M -Xms512M` | JVM heap settings |
| `wildfly_instance_count` | `3` | Number of co-located instances |
| `wildfly_ha_config` | `standalone-ha.xml` | Base WildFly config profile |
| `wildfly_port_range_offset_step` | `100` | HTTP/management port offset per instance |
| `wildfly_cluster_multicast_addr` | `230.0.0.4` | JGroups multicast address |
| `wildfly_app.name` | `helloworld.war` | Deployment artifact filename |
| `wildfly_app.local_path` | `/tmp/helloworld.war` | Path on Ansible controller |
| `wildfly_public_bind_addr` | `0.0.0.0` | Public bind address for each instance |

Override any variable at the command line:

```bash
ansible-playbook -i inventory playbook.yml \
  -e wildfly_version=40.0.0.Final \
  -e wildfly_instance_count=2 \
  -e wildfly_app.local_path=/my/app.war
```

---

## Roles Used

This demo exercises the following roles from the [middleware_automation.wildfly](https://github.com/ansible-middleware/wildfly) collection:

| Role | Purpose |
|:---|:---|
| [`wildfly_install`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_install/README.md) | Downloads and installs the WildFly archive |
| [`wildfly_systemd`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_systemd/README.md) | Creates a systemd unit per instance with the correct port offset |
| [`wildfly_app_deploy`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_app_deploy/README.md) | Deploys a WAR/EAR/JAR via jboss-cli |

### Additional roles available in the collection (not used in this demo)

| Role | Purpose |
|:---|:---|
| [`wildfly_driver`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_driver/README.md) | Install additional driver modules (e.g. JDBC) |
| [`wildfly_firewalld`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_firewalld/README.md) | Configure firewalld rules for all WildFly ports |
| [`wildfly_validation`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_validation/README.md) | Deep validation of a running WildFly instance |
| [`wildfly_uninstall`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_uninstall/README.md) | Remove a WildFly installation |
| [`wildfly_migration`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_migration/README.md) | Run the WildFly migration tool |
| [`wildfly_subs`](https://github.com/ansible-middleware/wildfly/blob/main/roles/wildfly_subs/README.md) | JBoss EAP RPM-based installation |

---

## Project Structure

```
wildfly-cluster-demo/
├── playbook.yml               # Main playbook
├── deploy_app.yml             # Per-instance app deploy helper
├── validate.yml               # Per-instance HTTP/management validation tasks
├── info.yml                   # Per-instance management API info tasks
├── inventory                  # Target host inventory
├── requirements.yml           # Collection dependencies (root)
├── vars/
│   └── vars.yml               # All tuneable demo variables
├── molecule/
│   └── default/
│       ├── molecule.yml       # Molecule scenario (Docker / UBI 9)
│       ├── requirements.yml   # Collection deps for Molecule
│       ├── prepare.yml        # Builds a minimal helloworld.war
│       └── verify.yml         # Smoke-tests all 3 instances
└── .github/
    └── workflows/
        └── ci.yml             # GitHub Actions – lint + molecule
```

---

## Running Tests (Molecule)

The Molecule scenario spins up a single UBI 9 container, installs WildFly with three instances, deploys the helloworld app, and verifies all HTTP / management endpoints.

```bash
# Install test dependencies
pip install "ansible-core>=2.16" "molecule>=24.2.0" "molecule-plugins[docker]>=23.5.0"
ansible-galaxy collection install -r molecule/default/requirements.yml

# Run the full test sequence
molecule test

# Or run individual steps
molecule create
molecule prepare   # builds helloworld.war on the controller
molecule converge  # runs playbook.yml
molecule verify    # smoke-tests endpoints
molecule destroy
```

---

## Using Red Hat JBoss EAP Instead of WildFly

The collection supports JBoss EAP (the Red Hat productised version of WildFly). Override the relevant variables:

```yaml
# vars/eap.yml
wildfly_version: '8.1.0'
wildfly_offline_install: true   # supply the zip via wildfly_archive_filename
wildfly_install_workdir: '/opt/eap/'
```

```bash
ansible-playbook -i inventory playbook.yml -e @vars/eap.yml
```

> **Note:** EAP requires a valid Red Hat subscription. Use the `middleware_automation.common` collection to handle credential-based download.

---

## References

- [middleware_automation.wildfly on Ansible Galaxy](https://galaxy.ansible.com/ui/repo/published/middleware_automation/wildfly/)
- [WildFly upstream collection on GitHub](https://github.com/ansible-middleware/wildfly)
- [WildFly Quickstarts](https://github.com/wildfly/quickstart)
- [WildFly Documentation](https://docs.wildfly.org/)
- [Ansible Middleware Project](https://ansible-middleware.github.io/)

---

## License

[GPL-2.0-only](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)

## Authors

- [Harsha Cherukuri](https://github.com/hcherukuri)
- Based on the [middleware_automation.wildfly](https://github.com/ansible-middleware/wildfly) collection by the Ansible Middleware team
