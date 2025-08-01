# 🔐 Task – SealedSecrets Automation

This project is a solution for the Instabug DevOps task.  
It automates the process of re-encrypting all SealedSecrets in a Kubernetes cluster using `kubeseal`.

---

## 🚀 Objective

The script scans all namespaces in the cluster for existing SealedSecrets, decrypts them using the original `private key`, then re-seals them with the updated `public key`.

---

## ⚙️ Tools & Technologies

- 🐧 Bash  
- ☸️ Kubernetes  
- 🔐 kubeseal (Bitnami SealedSecrets)  
- 🛠️ kubectl

---

## 📝 Usage

> Make sure you have the following:
- Access to the Kubernetes cluster
- `kubectl` installed and configured
- `kubeseal` installed
- Old certificate available as `old-cert.pem`

### 🔄 Run the script

```bash
chmod +x convert.sh
./convert.sh
```

---

## 📚 Full Plan & Technical Documentation

See [`docs/RE-ENCRYPTION_PLAN.md`](docs/RE-ENCRYPTION_PLAN.md) for a detailed explanation of the re-encryption mechanism, architecture, and error handling.

---

##  Author

Mahmoud Shiha  
[GitHub Profile](https://github.com/Mahmoud-shi7a)
