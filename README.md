# 🚀 Terraform Zero to Hero - AWS Edition

> **Master Terraform and AWS infrastructure from basics to production-ready deployments**

[![Terraform](https://img.shields.io/badge/Terraform-1.6+-623CE4?logo=terraform)](https://www.terraform.io/)
[![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?logo=amazon-aws)](https://aws.amazon.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

A comprehensive, hands-on course covering Terraform fundamentals to advanced production patterns with AWS. Perfect for DevOps engineers, cloud architects, and anyone looking to master Infrastructure as Code.

---

## 📚 Course Overview

This course takes you from Terraform basics to production-ready infrastructure management with AWS. Each chapter includes:
- 📖 **Comprehensive Theory** - Detailed explanations with diagrams
- 🧪 **Hands-on Labs** - Real-world AWS scenarios (3+ labs per chapter)
- 💼 **Interview Questions** - 15-20 questions with detailed answers per chapter
- ✅ **Best Practices** - Production-ready patterns and security considerations

**Total Duration:** 4-5 weeks with practice  
**Prerequisites:** Basic command line knowledge, AWS account (Free Tier)

---

## 📋 Chapters

| # | Chapter | Topics | Labs | Time |
|---|---------|--------|------|------|
| 1 | [Terraform Basics & HCL](./chapter-01/) | IaC, HCL syntax, workflow, state | 3 | 2 days |
| 2 | [AWS Fundamentals](./chapter-02/) | VPC, EC2, IAM, regions, AZs | 3 | 3 days |
| 3 | [State Management](./chapter-03/) | Remote state, locking, workspaces | 4 | 3 days |
| 4 | [Modules & Reusability](./chapter-04/) | Creating modules, registry, composition | 3 | 3 days |
| 5 | [Variables and Outputs](./chapter-05/) | Variable types, validation, outputs, locals | 3 | 2 days |
| 6 | [Provisioners & Null Resources](./chapter-06/) | local-exec, remote-exec, file provisioner | 3 | 2 days |
| 7 | [Data Sources & Functions](./chapter-07/) | Data sources, built-in functions, dynamic configs | 3 | 3 days |
| 8 | [Advanced Patterns](./chapter-08/) | for_each, dynamic blocks, lifecycle, workspaces | 3 | 3 days |
| 9 | [CI/CD Integration](./chapter-09/) | GitHub Actions, GitLab CI, Jenkins, testing | 3 | 3 days |
| 10 | [Production Best Practices](./chapter-10/) | Security, DR, monitoring, compliance | 3 | 4 days |

**Total:** 35+ hands-on labs, 150+ interview questions

---

## 🎯 Learning Paths

### 🟢 Beginner (0-1 year experience)
**Focus:** Chapters 1-4  
**Goal:** Master Terraform basics and AWS fundamentals
- Understand IaC principles
- Write HCL configurations
- Manage state effectively
- Create reusable modules

### 🟡 Intermediate (1-3 years experience)
**Focus:** Chapters 1-7  
**Goal:** Build production-ready infrastructure
- Advanced variable patterns
- Data sources and functions
- Provisioner usage
- Complex AWS architectures

### 🔴 Advanced/DevOps (3+ years experience)
**Focus:** All chapters  
**Goal:** Enterprise-grade infrastructure
- Advanced Terraform patterns
- CI/CD automation
- Security and compliance
- Disaster recovery
- Cost optimization

---

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/Ramiz-Takildar/terraform-zero-to-hero-aws.git
cd terraform-zero-to-hero-aws
```

### 2. Set Up Prerequisites

```bash
# Install Terraform
brew install terraform  # macOS
# or download from https://www.terraform.io/downloads

# Install AWS CLI
brew install awscli  # macOS
# or download from https://aws.amazon.com/cli/

# Configure AWS credentials
aws configure
```

### 3. Start Learning

```bash
# Open the checklist
open CHECKLIST.md

# Start with Chapter 1
cd chapter-01
cat README.md
```

### 4. Track Your Progress

Use [CHECKLIST.md](./CHECKLIST.md) to track your learning progress:

```bash
# Edit checklist
vim CHECKLIST.md

# Mark items complete: [ ] → [x]
# Commit your progress
git add CHECKLIST.md
git commit -m "Progress: Completed Chapter 1"
```

---

## 📖 How to Use This Course

### For Each Chapter:

1. **📚 Read Theory** - Start with `README.md` in each chapter
2. **🧪 Complete Labs** - Follow step-by-step instructions in `LABS.md`
3. **💼 Review Questions** - Study interview questions in `INTERVIEW.md`
4. **✅ Mark Progress** - Update `CHECKLIST.md` as you complete sections
5. **🔄 Practice** - Repeat labs, experiment with variations

### Study Tips:

- **Don't rush** - Understanding > Speed
- **Type everything** - Don't copy-paste, type to learn
- **Experiment** - Try variations of examples
- **Break things** - Learn by fixing errors
- **Document** - Keep notes of learnings
- **Review** - Revisit previous chapters

---

## 🎓 Certification Alignment

This course prepares you for:

| Certification | Chapters | Focus Areas |
|---------------|----------|-------------|
| **Terraform Associate** | 1-4 | Core workflow, HCL, state, modules |
| **AWS Solutions Architect Associate** | 1-7 | AWS services, networking, compute, storage |
| **AWS Solutions Architect Professional** | 1-10 | Advanced patterns, security, multi-account |
| **AWS DevOps Engineer Professional** | 1-10 | CI/CD, automation, monitoring |

---

## 💡 Key Features

### ✅ Comprehensive Coverage
- From basics to advanced production patterns
- Real-world AWS scenarios
- Security best practices throughout

### ✅ Hands-On Learning
- 35+ practical labs
- Step-by-step instructions
- Verification steps for each lab

### ✅ Interview Ready
- 150+ interview questions
- Detailed answers with examples
- Scenario-based questions

### ✅ Production Focus
- Security hardening
- Disaster recovery
- CI/CD integration
- Cost optimization
- Monitoring and alerting

---

## 🔧 Prerequisites

### Required:
- ✅ AWS Account (Free Tier sufficient)
- ✅ Basic command line knowledge
- ✅ Text editor (VS Code recommended)
- ✅ Git installed

### Recommended:
- 📚 Basic cloud computing concepts
- 📚 Understanding of networking basics
- 📚 Familiarity with Linux/Unix

### Tools to Install:
```bash
# Terraform
brew install terraform

# AWS CLI
brew install awscli

# Optional but helpful
brew install tflint      # Terraform linter
brew install tfsec       # Security scanner
brew install terraform-docs  # Documentation generator
```

---

## 💰 Cost Management

**Important:** All labs are designed to minimize costs:

- ✅ Use AWS Free Tier where possible
- ✅ Use `t2.micro` or `t3.micro` instances
- ✅ Destroy resources after each lab
- ✅ Set up billing alerts

### Cost Estimates:
- **Most labs:** $0 (Free Tier)
- **Some advanced labs:** $1-5 per lab
- **Monthly if not cleaned up:** $20-50

### Always Remember:
```bash
# After each lab
terraform destroy -auto-approve
```

---

## 📊 Progress Tracking

### Option 1: CHECKLIST.md (Recommended)
Track progress in Git:
```bash
vim CHECKLIST.md
# Mark items: [ ] → [x]
git commit -am "Progress update"
```

### Option 2: GitHub Issues
Create issues for each chapter and close as you complete them.

### Option 3: Personal Notes
Keep a learning journal with:
- What you learned
- Challenges faced
- Questions to research
- Key takeaways

---

## 🤝 Contributing

Found an error? Have a suggestion? Contributions are welcome!

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add improvement'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

---

## 📚 Additional Resources

### Official Documentation:
- [Terraform Documentation](https://www.terraform.io/docs)
- [AWS Documentation](https://docs.aws.amazon.com/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

### Community:
- [Terraform Community Forum](https://discuss.hashicorp.com/c/terraform-core)
- [AWS re:Post](https://repost.aws/)
- [r/Terraform](https://www.reddit.com/r/Terraform/)

### Tools:
- [Terraform Registry](https://registry.terraform.io/)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [VS Code Terraform Extension](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform)

---

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 👨‍💻 Author

**Ramiz Takildar**

- GitHub: [@Ramiz-Takildar](https://github.com/Ramiz-Takildar)
- Repository: [terraform-zero-to-hero-aws](https://github.com/Ramiz-Takildar/terraform-zero-to-hero-aws)

---

## ⭐ Show Your Support

If you find this course helpful, please:
- ⭐ Star this repository
- 🔄 Share with others
- 📝 Provide feedback
- 🐛 Report issues

---

## 🗺️ Roadmap

- [x] Chapters 1-10 complete
- [x] 35+ hands-on labs
- [x] 150+ interview questions
- [ ] Video tutorials (coming soon)
- [ ] Practice exams
- [ ] Advanced scenarios
- [ ] Multi-cloud examples

---

## 📞 Support

Need help? Have questions?

1. **Check the docs** - Each chapter has detailed explanations
2. **Review labs** - Step-by-step instructions included
3. **Open an issue** - For bugs or questions
4. **Community** - Join Terraform forums

---

## 🎉 Success Stories

Share your success story! Open an issue with:
- What you learned
- How it helped your career
- Certifications achieved
- Projects built

---

**Ready to become a Terraform expert? Start with [Chapter 1](./chapter-01/)!** 🚀

---

<div align="center">

**[⬆ Back to Top](#-terraform-zero-to-hero---aws-edition)**

Made with ❤️ for the DevOps community

</div>
