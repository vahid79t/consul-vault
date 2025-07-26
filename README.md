# Deploying Consul and Vault Using Docker Compose

This article demonstrates how to deploy **Consul** and **Vault** using Docker Compose. These tools are widely used in modern DevOps environments for service discovery and secrets management.

---

## 🔹 What is Consul?

**Consul** is a Service Mesh tool that provides:

- Service discovery  
- Health checks  
- Key-Value store  
- Node and service management via UI and CLI  

> While both Consul and Vault can store key-value pairs, **Vault** is focused on secure secrets management, whereas **Consul** emphasizes service discovery and distributed configuration.

---

## 🔹 What is Vault?

**Vault** is a secrets management system designed to:

- Securely store sensitive information such as passwords, tokens, and API keys  
- Manage access policies for secrets  
- Encrypt secrets at rest and in transit  

It supports various storage backends, including:

- Consul  
- Filesystem  
- Amazon S3  

---

## 🔸 Feature Comparison: Vault vs. Consul

_A zoomable image or table can be inserted here if needed._

---

## 🔹 Use Cases

- **Consul**: Ideal for service registration/discovery and key-value storage in microservice environments.  
- **Vault**: Best for managing application secrets, API keys, and credentials with fine-grained access controls.

---

## 🔹 Consul Setup Using Docker Compose

```yaml
version: '3.8'
services:
  consul:
    image: hashicorp/consul:latest
    container_name: consul
    restart: always
    volumes:
      - ./consul_data:/consul/data
      - ./consul_config:/consul/config
    ports:
      - "8500:8500"        # UI
      - "8600:8600/udp"    # DNS (UDP)
      - "8600:8600/tcp"    # DNS (TCP)
      - "8300:8300"        # Server RPC
      - "8301:8301"
      - "8301:8301/udp"
      - "8302:8302"
      - "8302:8302/udp"
    command: agent -server -bootstrap -ui -client=0.0.0.0 -data-dir=/consul/data -config-dir=/consul/config
    environment:
      - CONSUL_LOCAL_CONFIG={"acl":{"enabled":true, "default_policy":"deny", "down_policy":"extend-cache"}}
      - CONSUL_RETRY_JOIN_ADDRESS=178.18.63.42
      - CONSUL_GOSSIP_ENCRYPTION=enable
      - CONSUL_GOSSIP_ENCRYPTION_KEY=Mmqfj/p23iDGvBi1U6gE+WCr+dT+Fzm8+S2HcoVTB6c=
    networks:
      - consul-network

networks:
  consul-network:
    driver: bridge

volumes:
  consul_data:
  consul_config:
```

**Notes:**

- UI Access: [http://127.0.0.1:8500/ui](http://127.0.0.1:8500/ui)  
- `CONSUL_GOSSIP_ENCRYPTION_KEY` must be unique for secure clustering.  
- Ports:
  - 8500: Web UI  
  - 8600: DNS Interface (UDP & TCP)  
  - 8300: Server RPC  
  - 8301: Serf LAN (Gossip protocol)  

---

## 🔹 ACL System in Consul

With `acl.enabled=true`, all access is **denied by default**. You’ll need to bootstrap ACL:

```bash
docker exec -it consul consul acl bootstrap
```

**Example Output:**

```
AccessorID:       a3107a880712-0241
SecretID:         a92eb736-c4b7-940f
Description:      Bootstrap Token (Global Management)
Create Time:      2025-07-19T10:46:45Z
Policies:         global-management
```

> ⚠️ **Save the `SecretID` value** — it will be used to log into the Consul UI.

### Creating ACL Policies & Tokens

Define a policy (example):

```hcl
node_prefix "" {
  policy = "read"
}
service_prefix "" {
  policy = "write"
}
key_prefix "" {
  policy = "write"
}
```

Then assign this policy to a token.

---

## 🔹 Vault Setup Using Docker Compose

```yaml
version: '3.8'
services:
  vault:
    image: hashicorp/vault:latest
    container_name: vault-server
    ports:
      - "8200:8200"
    volumes:
      - ./vault-data:/vault/file
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_LOCAL_CONFIG: |
        {
          "storage": {
            "file": {
              "path": "/vault/file"
            }
          },
          "listener": [
            {
              "tcp": {
                "address": "0.0.0.0:8200",
                "tls_disable": true
              }
            }
          ],
          "default_lease_ttl": "168h",
          "max_lease_ttl": "720h",
          "ui": true
        }
    command: server
    restart: unless-stopped
```

**Notes:**

- UI Access: [http://127.0.0.1:8200](http://127.0.0.1:8200)  
- Vault data is stored in the local filesystem: `/vault/file`  
- In production, use **Consul backend** for HA and clustering.

---

## 🔹 Vault Initialization & Policy Management

When Vault starts for the first time, you’ll be asked to **initialize** and generate unseal keys.

> 💾 **Save the unseal keys and root token securely.**

---

### Creating a Policy in Vault

1. Create a policy file `editor.hcl`:

```hcl
path "secret/data/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

2. Copy the policy to the container:

```bash
docker cp editor.hcl vault-server:/tmp/editor.hcl
```

3. Enter the container:

```bash
docker exec -it vault-server /bin/sh
```

4. Set Vault environment:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

5. Login with root token:

```bash
vault login <your-token>
```

6. Write the policy:

```bash
vault policy write editor /tmp/editor.hcl
```

7. Create a token with the policy:

```bash
vault token create -policy=editor -ttl=1h
```

---

## ✅ Conclusion

This guide explained how to:

- Deploy **Consul** and **Vault** using Docker Compose  
- Configure **ACL** and **encryption** in Consul  
- Initialize Vault and create **custom policies**

These tools are essential for secure infrastructure management and service discovery in modern distributed systems.

---

**Author:** [Your Name or GitHub Handle]  
**Date:** July 2025
