# AWS How-To Guides 🚀

> Hands-on, PoC-driven AWS guides for engineers — from beginner to advanced.
> Each module is structured to help you **understand the concept**, **run the PoC**, and **extend to real-world use cases**.

---

## 📚 Modules

| Module | Topics Covered | Difficulty |
|--------|---------------|------------|
| [IAM](./IAM/README.md) | Users, Groups, Roles, Policies, Cross-Account, Permission Boundaries, ABAC | 🟢 Beginner → 🔴 Advanced |
| [VPC](./VPC/README.md) | Subnets, Route Tables, Security Groups, NACLs, NAT GW, Peering, Endpoints, Transit Gateway | 🟢 Beginner → 🔴 Advanced |

> More modules coming soon: VPC, EC2, S3, EKS, Lambda, Organizations, etc.

---

## 🧰 Prerequisites

Before starting any lab, ensure you have the following tools installed:

```bash
# AWS CLI v2
aws --version

# Optional but recommended
jq --version        # JSON parsing
terraform --version # For IaC labs
```

- An AWS account (Free Tier works for most labs)
- AWS CLI configured with admin or appropriate permissions
- Basic understanding of Linux CLI

---

## 🗂️ How Labs Are Structured

Each lab follows this format:

```
📁 module/
├── README.md          ← Module overview & index
└── XX-lab-name/
    ├── README.md      ← Lab goal, concepts, architecture
    ├── commands.sh    ← All CLI commands (copy-paste ready)
    └── policies/      ← JSON policy documents
```

---

## ⚡ Quick Start

```bash
git clone https://github.com/naveenramasamy11/aws-how-to-guides.git
cd aws-how-to-guides/IAM
```

Pick a lab, read the README, and follow along!

---

## 🤝 Contributing

Pull requests are welcome! Please open an issue first to discuss what you'd like to add or change.

---

## 📝 License

MIT License — see [LICENSE](./LICENSE) for details.
