# Data Security, Data Privacy, and Regulatory Requirements: Three Pillars of Data Governance

As organizations increasingly rely on data to drive decisions, products, and services, three terms appear repeatedly in technical and governance discussions: **data security**, **data privacy**, and **regulatory requirements**. 
This article explains each concept, how they differ and how they overlap


---

## 1. Data Security

### What Is Data Security?

**Data security protects data from unauthorized access, modification, deletion, or disclosure.**
It ensures data remains **confidential, accurate, and available** to authorized users. While access control is critical, additional mechanisms such as encryption, integrity checks, and monitoring address risks like insider threats, system breaches, or network interception.

**Importantly, data security ensures that data is available for the authorized users while blocking unauthorized parties.**

---

### Core Objectives (The CIA Triad)

1. **Confidentiality** – Only authorized users or systems can access data.
2. **Integrity** – Data remains accurate, complete, and unaltered.
3. **Availability** – Authorized users can access data when needed.

---

### Key Data Security Mechanisms

| Mechanism                             | How It Works                                                                                                | CIA Element(s)                     | Example / Use Case                                                                 |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------- |
| **Access Control**                    | Authentication (passwords, MFA, IAM roles, Security Tokens etc) and authorization (RBAC (Role Based Access Control)/ABAC (Attribute Based Access Control),FGAC (Fine-Grained Access Control), least privilege)                              | Confidentiality & partly Integrity | Employees can only access relevant records.                                        |
| **Encryption (at rest & in transit)** | Converts data into unreadable form for unauthorized users; authorized users can decrypt and use it normally | Confidentiality & partly Integrity | Database encryption protects against stolen backups; TLS protects data in transit. |
| **Integrity Protection**              | Uses hashes, checksums, digital signatures, or constraints                                                  | Integrity                          | Hash mismatch alerts system to tampering.                                          |
| **Monitoring & Logging**              | Tracks access and modifications                                                                             | Confidentiality & Integrity        | Audit logs detect brute-force login attempts.                                      |
| **Availability Controls**             | Backups, redundancy, failover, DDoS protection                                                              | Availability                       | Replicated databases ensure uptime.                                                |
| **Masking / Tokenization**            | Obfuscates sensitive data for non-production use                                                            | Confidentiality                    | Masking and tokenization allow organizations to use sensitive data safely by replacing the real values with obfuscated or tokenized equivalents. This enables teams to develop and test applications without exposing actual personal or payment data, perform analytics and reporting while keeping sensitive fields hidden, and share information with third parties or support teams without revealing real PII, PCI, or PHI.

---

### Why Encryption Is Essential Beyond Access Control

Access control determines **who is allowed** to access data—but it cannot fully prevent misuse if credentials are stolen, insiders act maliciously, or a system is breached. Encryption provides a “last line of defense”:

* **Mitigates Insider Threats** – Even DBAs cannot read sensitive fields without keys.
* **Protects Data in Transit** – Prevents interception or man-in-the-middle attacks.
* **Reduces Breach Impact** – Stolen backups are useless without decryption keys.
* **Maintains Usability for Authorized Users** – Data is decrypted on access for legitimate use.
* **Regulatory Compliance** – Many laws (PCI DSS, GDPR, HIPAA) mandate encryption of sensitive data.

---

## 2. Data Privacy

### What Is Data Privacy?

**Data privacy concerns personal data**—information relating to identified or identifiable individuals. It governs the **entire lifecycle**, from collection to deletion, and defines the **rights individuals have over their information**.

---

### Security vs. Privacy: The Key Distinction

Strong security does not guarantee privacy.

* **Example:** Health data is encrypted (Security) but sold to an advertiser without consent → Privacy violation.
* **Rule:** Security protects data; Privacy governs its authorized use.

---

### Core Privacy Principles

* **Purpose Limitation:** Use data only for the reason it was collected.
* **Data Minimization:** Collect only what is necessary.
* **Storage Limitation:** Delete data once it is no longer needed.
* **Individual Rights:** Right to be forgotten, right to portability, etc.

---

## 3. Compliance


### What Are Regulatory Requirements?

**Regulatory requirements are laws or legally binding standards issued by governments.**
They define **concrete obligations** for handling data. While many focus on personal data, regulations often extend to financial records, operational resilience, and **data sovereignty** (where data is stored).

---

### Key Regulations (Overview)

| **Regulation**                                                                    | **Scope**                       | **Key Focus**                                                                          |
| --------------------------------------------------------------------------------- | ------------------------------- | -------------------------------------------------------------------------------------- |
| **GDPR (General Data Protection Regulation)**                                     | Personal Data (EU & EEA)        | Individual rights and lawful processing of personal data (e.g., Right to be Forgotten) |
| **CCPA / CPRA (California Consumer Privacy Act / California Privacy Rights Act)** | Personal Data (California, USA) | Consumer privacy rights — access, deletion, opt‑out of sharing/sale                    |
| **HIPAA (Health Insurance Portability and Accountability Act)**                   | Healthcare Data (USA)           | Privacy and technical security standards for protected health information (PHI)        |
| **PCI DSS (Payment Card Industry Data Security Standard)**                        | Payments & Cardholder Data      | Security standards (including encryption) for payment card data                        |
| **DORA (Digital Operational Resilience Act)**                                     | Financial Sector ICT (EU)       | Operational and cyber resilience for financial ICT systems                             |
| **SOX (Sarbanes–Oxley Act)**                                                      | Financial Reporting (USA)       | Integrity, accuracy, and auditability of corporate financial records                   |


### Regulatory Compliance: The Final Audit

Compliance is the state of Data Security and Privacy implementation meeting specific Regulatory Requirements. Compliance is the "passed exam" of the data world. It is the process of proving to an external authority (a regulator, a partner, or a customer) that your Data Protection practices are actually working and align with the law.

### Note on Data Protection

Data Protection is the use of Data Security to meet the requirements of Data Privacy.It is the operational practice of ensuring data is handled safely, stays available, and remains under the control of its rightful owner. While Privacy is the intent and Security is the mechanism, Protection is the implementation.


## 4. Data Quality
Data Quality ensures data is **useful, correct, and formatted properly**. While Data Security ensures data isn’t tampered with , Quality ensures it is accurate and fit for purpose. It is the “Value Layer.”

**Dimensions of Data Quality:**

1. **Accuracy** – Does the data reflect real-world values correctly?
2. **Completeness** – Is all expected data present?
3. **Consistency** – Are there contradictions within or across datasets?
4. **Uniqueness** – Are there duplicate records where there shouldn’t be?
5. **Integrity** – Are relationships between entities maintained?
6. **Validity** – Do values conform to formats, domains, and rules?
7. **Availability** – Is data delivered on time and according to SLA?

## 5. The Bigger Picture: Data Governance


1. **Privacy** defines *what* can be done with data.
2. **Security** enforces those rules through technical controls.
3. **Complaince to the Regulations/Laws** provide the framework and accountability for both.
4. **Data Quality** : Data Quality ensures the data is useful, correct, and formatted properly. 

### Quick Win: Mental Models
**1. Healthcare Example**
* **Data Security** – Hospital IT systems, firewalls, encryption, and access controls ensure only authorized staff can access patient records.
* **Data Privacy** – Patients give consent; the hospital enforces policies defining which staff can access data for treatment, billing, or research.
* **Regulatory Requirements** – HIPAA auditors ensure compliance with privacy and security laws.

**2. Online Retail Example**

* **Data Security** – Secure servers, SSL-encrypted payment pages, and access-controlled databases prevent unauthorized access to customer information.
* **Data Privacy** – Customers provide consent; the retailer enforces policies on who can access data for orders, marketing, or analytics.
* **Regulatory Requirements** – PCI DSS and GDPR/CCPA audits verify secure and legal handling of customer data.

**3. Banking Example**

* **Data Security** – Encryption, multi-factor authentication, monitoring, and firewalls prevent unauthorized access to accounts.
* **Data Privacy** – Policies and consent agreements define which third-party services can access financial data for operations, credit checks, or reporting.
* **Regulatory Requirements** – Banking regulators ensure compliance with privacy, anti-fraud, and KYC rules.

**Takeaway:** Security protects data technically, privacy governs proper use and consent, and regulations enforce accountability.


