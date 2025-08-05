# 🔐 Ansible + HashiCorp Vault Integration (AppRole Auth) – PoC

This guide demonstrates how to integrate [HashiCorp Vault](https://www.vaultproject.io/) with [Ansible](https://www.ansible.com/) using the AppRole authentication method. It includes all steps from installing Vault on an EC2 instance to securely retrieving secrets during an Ansible playbook run.

---

## 🧰 Prerequisites

- Amazon Linux 2 EC2 Instance
- Python 3.9+
- Ansible installed
- Internet access to install packages
- Port `8200` opened in the EC2 Security Group for Vault Web UI access (optional)

---

## 📦 Step 1: Install Vault Server on Amazon Linux

```bash
sudo yum install -y yum-utils shadow-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install vault
```

---

## 🚀 Step 2: Start Vault Server in Dev Mode

```bash
vault server -dev -dev-listen-address="0.0.0.0:8200"
```

> 💡 Note: This is only for PoC/dev. Do **NOT** use `-dev` mode in production.

Access the UI via: `http://<your-ec2-public-ip>:8200`

---

## 🛡️ Step 3: Set Up Vault Secrets, Policy & AppRole

### 3.1: Export VAULT_ADDR

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

### 3.2: Store a Sample Secret

```bash
vault kv put secret/myapp db_user="admin" db_password="SuperSecret123"
```

### 3.3: Create Policy File (`ansible-policy.hcl`)

```hcl
path "secret/data/myapp" {
  capabilities = ["read"]
}
```

### 3.4: Apply Policy

```bash
vault policy write ansible-policy ansible-policy.hcl
```

### 3.5: Enable AppRole Auth (if not enabled)

```bash
vault auth enable approle
```

### 3.6: Create AppRole for Ansible

```bash
vault write auth/approle/role/ansible-role   secret_id_ttl=60m   token_num_uses=10   token_ttl=60m   token_max_ttl=120m   policies=ansible-policy
```

---

## 🔑 Step 4: Retrieve AppRole Credentials

### 4.1: Get `role_id`

```bash
vault read auth/approle/role/ansible-role/role-id
```

### 4.2: Generate `secret_id`

```bash
vault write -f auth/approle/role/ansible-role/secret-id
```

Copy and save both values for the Ansible playbook.

---

## 🐍 Step 5: Install HVAC – Python Vault API Client

```bash
/usr/bin/python3.9 -m pip install --ignore-installed hvac
```

---

## 📁 Step 6: Install Required Ansible Collections

```bash
ansible-galaxy collection install community.hashi_vault
```

---

## 📜 Step 7: Create Ansible Playbook

### File: `ansible.yml`

```yaml
- hosts: localhost
  gather_facts: false
  vars:
    vault_addr: "http://<your-ec2-ip>:8200"
    vault_role_id: "<your-role-id>"
    vault_secret_id: "<your-secret-id>"

  tasks:
    - name: Fetch database password from Vault
      debug:
        msg: "{{ lookup('community.hashi_vault.hashi_vault',
                        'secret=secret/data/myapp:db_password',
                        url=vault_addr,
                        auth_method='approle',
                        role_id=vault_role_id,
                        secret_id=vault_secret_id) }}"
```

> 🔐 Replace `<your-ec2-ip>`, `<your-role-id>`, and `<your-secret-id>` with actual values.

---

## ✅ Step 8: Run the Playbook

```bash
ansible-playbook ansible.yml
```

Expected output:

```yaml
TASK [Fetch database password from Vault]
ok: [localhost] => {
    "msg": "SuperSecret123"
}
```

---

## 📁 Project Directory Structure

```
ansible-vault-integration/
├── ansible.yml
├── ansible-policy.hcl
└── README.md
```

---

## 📌 Notes

- This setup uses Vault dev mode – suitable only for PoC/demo purposes.
- For production:
  - Use Vault in HA mode with proper storage backend.
  - Enable TLS on Vault.
  - Store secrets under proper namespaces.
  - Avoid hardcoding credentials – use environment variables or Ansible Vault.

---

## 📚 References

- [Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [Ansible HashiCorp Vault Collection](https://docs.ansible.com/ansible/latest/collections/community/hashi_vault/)
