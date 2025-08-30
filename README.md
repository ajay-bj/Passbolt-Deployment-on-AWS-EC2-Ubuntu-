# 🚀 Passbolt Deployment on AWS EC2 (Ubuntu)

This repository documents the steps to deploy **Passbolt** (Open Source Password Manager) on an **AWS EC2 Ubuntu instance** using **Docker & Docker Compose**.

---

## 📌 Prerequisites
- An AWS EC2 instance (Ubuntu 22.04 recommended)
- Security Group with ports **80 (HTTP)** and **443 (HTTPS)** open
- Docker & Docker Compose installed

---

## ⚡ Step 1: Update System
```bash
sudo apt update && sudo apt upgrade -y
