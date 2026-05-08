# 🛒 Ansible Ecommerce Project

A fully automated, infrastructure-as-code deployment of a microservices-based ecommerce application (**RoboShop**) on AWS using **Ansible**. This project provisions EC2 instances, configures Route 53 DNS records, and deploys all application services end-to-end with zero manual intervention.

---

## 📐 Architecture Overview

The application follows a microservices architecture where each service runs on a dedicated EC2 instance, all registered under a private domain (`ellamma.fun`) via AWS Route 53.

```
                        ┌─────────────────────┐
                        │      Internet        │
                        └────────┬────────────┘
                                 │ (Public IP)
                        ┌────────▼────────────┐
                        │  Frontend (Nginx)    │
                        │  ellamma.fun         │
                        └────────┬────────────┘
                                 │
          ┌──────────────────────┼─────────────────────────┐
          │                      │                          │
   ┌──────▼──────┐     ┌─────────▼──────┐       ┌──────────▼──────┐
   │  Catalogue  │     │     User        │       │     Cart         │
   │  (Node.js)  │     │  (Node.js)      │       │   (Node.js)      │
   └──────┬──────┘     └────────────────┘       └──────────────────┘
          │
   ┌──────▼──────┐     ┌─────────────────┐      ┌──────────────────┐
   │  MongoDB    │     │     MySQL        │      │     Redis         │
   │  (Database) │     │   (Database)     │      │    (Cache)        │
   └─────────────┘     └─────────────────┘      └──────────────────┘

   ┌─────────────┐     ┌─────────────────┐
   │  Shipping   │     │    Payment       │
   │   (Java)    │     │   (Python)       │
   └─────────────┘     └─────────────────┘

   ┌─────────────┐
   │  RabbitMQ   │
   │  (Messaging) │
   └─────────────┘
```

---

## 🧩 Services

| Service | Technology | Domain | Role |
|---|---|---|---|
| **Frontend** | Nginx | `ellamma.fun` | Reverse proxy & UI serving |
| **Catalogue** | Node.js | `catalogue.ellamma.fun` | Product catalog API |
| **User** | Node.js | `user.ellamma.fun` | User management |
| **Cart** | Node.js | `cart.ellamma.fun` | Shopping cart |
| **Shipping** | Java | `shipping.ellamma.fun` | Shipping & delivery |
| **Payment** | Python | `payment.ellamma.fun` | Payment processing |
| **MongoDB** | MongoDB | `mongodb.ellamma.fun` | NoSQL database |
| **MySQL** | MySQL | `mysql.ellamma.fun` | Relational database |
| **Redis** | Redis | `redis.ellamma.fun` | In-memory cache |
| **RabbitMQ** | RabbitMQ | `rabbitmq.ellamma.fun` | Message broker |

---

## 📁 Project Structure

```
Ansible_Ecommerce_Project/
│
├── create-ec2-r53-ansible.yaml   # Provision EC2 instances + Route 53 DNS records
├── inventory.ini                  # Ansible inventory — all service hosts
│
├── frontend.yaml                  # Deploy Nginx frontend
├── catalogue.yaml                 # Deploy Catalogue service
├── user.yaml                      # Deploy User service
├── cart.yaml                      # Deploy Cart service
├── shipping.yaml                  # Deploy Shipping service
├── payment.yaml                   # Deploy Payment service
│
├── mongodb.yaml                   # Deploy MongoDB
├── mysql.yaml                     # Deploy MySQL
├── redis.yaml                     # Deploy Redis
├── rabbitmq.yaml                  # Deploy RabbitMQ
│
├── nginx.conf                     # Custom Nginx reverse proxy config
├── mongo.repo                     # MongoDB YUM repository definition
├── rabbitmq.repo                  # RabbitMQ YUM repository definition
│
├── cart.service                   # systemd unit — Cart
├── catalogue.service              # systemd unit — Catalogue
├── payment.service                # systemd unit — Payment
├── shipping.service               # systemd unit — Shipping
└── user.service                   # systemd unit — User
```

---

## ✅ Prerequisites

Before running the playbooks, make sure you have the following:

- **Ansible** installed on your control node (`ansible-core >= 2.12`)
- **AWS CLI** configured with appropriate IAM credentials
- **Boto3** Python library installed (`pip install boto3`)
- An AWS account with:
  - A VPC and Security Group (`sg-020b146d0d1696186`)
  - A Route 53 hosted zone for your domain
  - Appropriate IAM permissions to create EC2 instances and Route 53 records
- The `amazon.aws` Ansible collection installed:
  ```bash
  ansible-galaxy collection install amazon.aws
  ```

---

## ⚙️ Configuration

Before running the playbooks, update the variables in `create-ec2-r53-ansible.yaml` to match your environment:

```yaml
vars:
  image_id: ami-0220d79f3f480ecf5   # Your AMI ID (Amazon Linux 2)
  zone_id: "YOUR_ROUTE53_ZONE_ID"   # Your Route 53 Hosted Zone ID
  sg_id: sg-XXXXXXXXXXXXXXXXX        # Your Security Group ID
  domain_name: your-domain.com       # Your registered domain
```

Also update `inventory.ini` to reflect your domain:

```ini
[all:vars]
ansible_user=ec2-user
ansible_password=YOUR_PASSWORD
```

> ⚠️ **Security Note:** Avoid storing passwords in plaintext. Use [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) to encrypt sensitive values.

---

## 🚀 Deployment

### Step 1 — Provision Infrastructure

Create all EC2 instances and their Route 53 DNS records in one shot:

```bash
ansible-playbook create-ec2-r53-ansible.yaml -e "instance=[mongodb,catalogue,redis,user,cart,mysql,shipping,rabbitmq,payment,frontend]"
```

This playbook will:
- Launch a `t3.micro` EC2 instance per service
- Register each instance with a private DNS A record (e.g., `catalogue.ellamma.fun`)
- Register the frontend with its **public** IP at the root domain

### Step 2 — Deploy All Services

Run the individual playbooks against the inventory. You can deploy services independently or chain them:

```bash
# Databases first (no dependencies)
ansible-playbook -i inventory.ini mongodb.yaml
ansible-playbook -i inventory.ini mysql.yaml
ansible-playbook -i inventory.ini redis.yaml
ansible-playbook -i inventory.ini rabbitmq.yaml

# Backend services
ansible-playbook -i inventory.ini catalogue.yaml
ansible-playbook -i inventory.ini user.yaml
ansible-playbook -i inventory.ini cart.yaml
ansible-playbook -i inventory.ini shipping.yaml
ansible-playbook -i inventory.ini payment.yaml

# Frontend last
ansible-playbook -i inventory.ini frontend.yaml
```

### Recommended Deployment Order

```
MongoDB → MySQL → Redis → RabbitMQ
  → Catalogue → User → Cart → Shipping → Payment
    → Frontend
```

---

## 🔍 How It Works

### Infrastructure Provisioning (`create-ec2-r53-ansible.yaml`)

The provisioning playbook runs on `localhost` and uses the `amazon.aws` collection to:
1. Create EC2 instances in a loop for each service name passed in
2. Register private IP addresses as Route 53 A records for backend services
3. Register the **public** IP of the `frontend` instance at the root domain

### Service Deployment

Each service playbook (e.g., `catalogue.yaml`) connects to the appropriate host group in `inventory.ini` and:
1. Installs the required runtime (Node.js, Java, Python, etc.)
2. Downloads the application artifact from S3
3. Configures the service using systemd unit files (`.service` files in the repo)
4. Starts and enables the service

### Frontend (`frontend.yaml`)

The frontend playbook:
1. Disables the old Nginx package and enables Nginx 1.24
2. Downloads and extracts the RoboShop frontend bundle from S3
3. Replaces the default Nginx config with the custom `nginx.conf` (which acts as a reverse proxy to backend services)
4. Restarts Nginx

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Provisioning | Ansible + AWS (`amazon.aws` collection) |
| Cloud | AWS EC2, Route 53 |
| Web Server | Nginx 1.24 |
| Runtimes | Node.js, Java, Python |
| Databases | MongoDB, MySQL |
| Cache | Redis |
| Messaging | RabbitMQ |
| OS | Amazon Linux 2 (RHEL-based) |

---

## 📝 Notes

- All backend services communicate using **private DNS** within the VPC — no hardcoded IPs needed.
- The `frontend` instance is the only one exposed to the public internet via its **public IP**.
- The `nginx.conf` acts as the API gateway, routing `/api/<service>` paths to the appropriate backend.
- systemd `.service` files ensure services restart automatically on instance reboot.

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

*Built with ❤️ using Ansible and AWS*
