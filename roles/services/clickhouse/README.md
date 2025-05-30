# Ansible Role: ClickHouse

## Description

This Ansible role automates the installation, configuration, and management of the ClickHouse server.  
It installs ClickHouse from the official Yandex repository, manages configuration files, handles the ClickHouse service, sets up users and permissions, and supports cluster configuration with ZooKeeper integration.

---

## Features

- Installs ClickHouse from the official Yandex stable repository  
- Configures main configuration files: `config.xml`, `users.xml`, and `zookeeper.xml`  
- Manages the `clickhouse-server` service (start, restart, enable at boot)  
- Creates and configures ClickHouse users with passwords and access rights  
- Supports cluster setup with ZooKeeper nodes configuration  
- Compatible with Debian/Ubuntu and CentOS/RHEL systems  

---

## Requirements

- Ansible 2.9 or higher  
- Linux distributions: Ubuntu 20.04+, Debian 10+, CentOS 7/8  
- Sudo privileges on target hosts  

Support
If you find any bugs, please open an issue.
Pull requests for improvements and features are welcome.



License
MIT License © KasOps

Useful Links
ClickHouse Official Documentation

ClickHouse GitHub Repository

Ansible Documentation

