# SSH ম্যানেজমেন্ট ও রিকভারি সম্পূর্ণ গাইড

## Ceph ক্লাস্টার ও OpenStack ইনস্ট্যান্সের জন্য প্রফেশনাল ডকুমেন্টেশন

---

**ডকুমেন্ট ভার্শন:** 1.0  
**শেষ আপডেট:** মার্চ ২০২৬  
**লেখক:** Sumon - IT Infrastructure & DevOps Specialist  
**ভাষা:** বাংলা  
**পড়ার সময়:** আনুমানিক ৯০ মিনিট  

---

## সূচিপত্র

### [ভূমিকা](#ভূমিকা)

### [অংশ ১: SSH প্রযুক্তির মৌলিক ধারণা](#অংশ-১-ssh-প্রযুক্তির-মৌলিক-ধারণা)
- [১.১ SSH কী এবং কেন প্রয়োজন](#১১-ssh-কী-এবং-কেন-প্রয়োজন)
- [১.২ SSH এর ইতিহাস ও বিবর্তন](#১২-ssh-এর-ইতিহাস-ও-বিবর্তন)
- [১.৩ SSH আর্কিটেকচার ও কাজের পদ্ধতি](#১৩-ssh-আর্কিটেকচার-ও-কাজের-পদ্ধতি)
- [১.৪ SSH Key Types এবং তাদের বৈশিষ্ট্য](#১৪-ssh-key-types-এবং-তাদের-বৈশিষ্ট্য)
- [১.৫ Authentication Methods](#১৫-authentication-methods)

### [অংশ ২: Ceph ক্লাস্টারে Passwordless SSH সেটআপ](#অংশ-২-ceph-ক্লাস্টারে-passwordless-ssh-সেটআপ)
- [২.১ প্রয়োজনীয় প্রস্তুতি ও পরিকল্পনা](#২১-প্রয়োজনীয়-প্রস্তুতি-ও-পরিকল্পনা)
- [২.২ পুরনো SSH কনফিগারেশন ক্লিনআপ](#২২-পুরনো-ssh-কনফিগারেশন-ক্লিনআপ)
- [২.৩ ইউজার ম্যানেজমেন্ট ও পারমিশন](#২৩-ইউজার-ম্যানেজমেন্ট-ও-পারমিশন)
- [২.৪ SSH Key জেনারেশন ও ম্যানেজমেন্ট](#২৪-ssh-key-জেনারেশন-ও-ম্যানেজমেন্ট)
- [২.৫ Key Distribution পদ্ধতি](#২৫-key-distribution-পদ্ধতি)
- [২.৬ SSH Config ফাইল কনফিগারেশন](#২৬-ssh-config-ফাইল-কনফিগারেশন)
- [২.৭ পারমিশন ম্যানেজমেন্ট](#২৭-পারমিশন-ম্যানেজমেন্ট)
- [২.৮ ভেরিফিকেশন ও টেস্টিং](#২৮-ভেরিফিকেশন-ও-টেস্টিং)

### [অংশ ৩: SSH কনফিগারেশন ট্রাবলশুটিং](#অংশ-৩-ssh-কনফিগারেশন-ট্রাবলশুটিং)
- [৩.১ AllowUsers সমস্যা ও সমাধান](#৩১-allowusers-সমস্যা-ও-সমাধান)
- [৩.২ Authentication Methods কনফ্লিক্ট](#৩২-authentication-methods-কনফ্লিক্ট)
- [৩.৩ Permission Denied সমস্যা](#৩৩-permission-denied-সমস্যা)
- [৩.৪ Connection Refused সমস্যা](#৩৪-connection-refused-সমস্যা)
- [৩.৫ কাস্টম কনফিগারেশন ফাইল সমস্যা](#৩৫-কাস্টম-কনফিগারেশন-ফাইল-সমস্যা)
- [৩.৬ লগ অ্যানালাইসিস ও ডিবাগিং](#৩৬-লগ-অ্যানালাইসিস-ও-ডিবাগিং)

### [অংশ ৪: OpenStack ইনস্ট্যান্স SSH রিকভারি](#অংশ-৪-openstack-ইনস্ট্যান্স-ssh-রিকভারি)
- [৪.১ হারানো KeyPair সমস্যা](#৪১-হারানো-keypair-সমস্যা)
- [৪.২ OpenStack Console ব্যবহার](#৪২-openstack-console-ব্যবহার)
- [৪.৩ Root Password রিসেট](#৪৩-root-password-রিসেট)
- [৪.৪ Password SSH Enable করা](#৪-password-ssh-enable-করা)
- [৪.৫ নতুন KeyPair সেটআপ](#৪৫-নতুন-keypair-সেটআপ)
- [৪.৬ Security Group ও Firewall কনফিগারেশন](#৪৬-security-group-ও-firewall-কনফিগারেশন)

### [অংশ ৫: নিরাপত্তা ও Best Practices](#অংশ-৫-নিরাপত্তা-ও-best-practices)
- [৫.১ SSH Security Hardening](#৫১-ssh-security-hardening)
- [৫.২ Key Management Best Practices](#৫২-key-management-best-practices)
- [৫.৩ Access Control ও Audit](#৫৩-access-control-ও-audit)
- [৫.৪ Backup ও Disaster Recovery](#৫৪-backup-ও-disaster-recovery)
- [৫.৫ Compliance ও Regulatory Requirements](#৫৫-compliance-ও-regulatory-requirements)

### [অংশ ৬: অপারেশনাল গাইড](#অংশ-৬-অপারেশনাল-গাইড)
- [৬.১ দৈনিক অপারেশন](#৬১-দৈনিক-অপারেশন)
- [৬.২ মাসিক রক্ষণাবেক্ষণ](#৬২-মাসিক-রক্ষণাবেক্ষণ)
- [৬.৩ Key Rotation প্রসিডিউর](#৬৩-key-rotation-প্রসিডিউর)
- [৬.৪ Monitoring ও Alerting](#৬-monitoring-ও-alerting)
- [৬.৫ Documentation Management](#৬৫-documentation-management)

### [পরিশিষ্ট](#পরিশিষ্ট)
- [পরিশিষ্ট A: SSH কমান্ড রেফারেন্স](#পরিশিষ্ট-a-ssh-কমান্ড-রেফারেন্স)
- [পরিশিষ্ট B: কনফিগারেশন ফাইল রেফারেন্স](#পরিশিষ্ট-b-কনফিগারেশন-ফাইল-রেফারেন্স)
- [পরিশিষ্ট C: ট্রাবলশুটিং চেকলিস্ট](#পরিশিষ্ট-c-ট্রাবলশুটিং-চেকলিস্ট)
- [পরিশিষ্ট D: নিরাপত্তা চেকলিস্ট](#পরিশিষ্ট-d-নিরাপত্তা-চেকলিস্ট)

---

## ভূমিকা

### ডকুমেন্টের উদ্দেশ্য

এই ডকুমেন্টটি তৈরি করা হয়েছে বাস্তব বিশ্বের IT Infrastructure এবং DevOps পরিবেশে SSH (Secure Shell) ম্যানেজমেন্টের সম্পূর্ণ গাইডলাইন হিসেবে। বিশেষ করে Ceph স্টোরেজ ক্লাস্টার এবং OpenStack ক্লাউড ইনফ্রাস্ট্রাকচার ম্যানেজমেন্টের ক্ষেত্রে SSH কীভাবে সঠিকভাবে সেটআপ, কনফিগার, ট্রাবলশুট এবং মেইনটেইন করতে হয়, তার বিস্তারিত আলোচনা করা হয়েছে।

### পাঠকের উদ্দেশ্য

এই ডকুমেন্টটি মূলত নিম্নলিখিত পেশাদারদের জন্য তৈরি:

- **সিস্টেম অ্যাডমিনিস্ট্রেটর**: যারা Linux/Unix সার্ভার ম্যানেজ করেন
- **DevOps ইঞ্জিনিয়ার**: যারা ক্লাউড ইনফ্রাস্ট্রাকচার ও অটোমেশন নিয়ে কাজ করেন
- **নেটওয়ার্ক অ্যাডমিনিস্ট্রেটর**: যারা নেটওয়ার্ক সিকিউরিটি ও এক্সেস ম্যানেজমেন্ট দেখেন
- **ক্লাউড আর্কিটেক্ট**: যারা OpenStack, AWS, Azure প্ল্যাটফর্ম ডিজাইন করেন
- **IT সিকিউরিটি স্পেশালিস্ট**: যারা ইনফ্রাস্ট্রাকচার সিকিউরিটি নিশ্চিত করেন

### বাস্তব অভিজ্ঞতার আলোকে

এই ডকুমেন্টের প্রতিটি অধ্যায় বাস্তব প্রজেক্টের অভিজ্ঞতা থেকে তৈরি। উদাহরণস্বরূপ, একটি ৩-নোড Ceph ক্লাস্টার সেটআপ করার সময় আমরা যে সমস্যাগুলোর সম্মুখীন হয়েছিলাম - যেমন AllowUsers কনফিগারেশন সমস্যা, Authentication Methods কনফ্লিক্ট, OpenStack ইনস্ট্যান্সে KeyPair হারিয়ে ফেলা - এসব বাস্তব সমস্যার সমাধান এখানে বিস্তারিত আলোচনা করা হয়েছে।

**বাস্তব গল্প:** ২০২৪ সালের ডিসেম্বরে একটি প্রোডাকশন এনভায়রনমেন্টে Ceph ক্লাস্টার ডেপ্লয় করার সময়, আমাদের টিম একটি গুরুতর সমস্যার সম্মুখীন হয়। তিনটি নোডের মধ্যে SSH কানেক্টিভিটি সেটআপ করার সময়, দ্বিতীয় নোডে (ceph2) বারবার "Permission denied" এরর আসছিল। প্রায় ৪ ঘণ্টা ডিবাগিং করার পর我们发现 যে `/etc/ssh/sshd_config.d/50-cloud-init.conf` ফাইলে `AllowUsers ceph2` সেট করা ছিল, যা শুধুমাত্র ceph2 ইউজারকেই অনুমতি দিচ্ছিল। এই বাস্তব অভিজ্ঞতা থেকেই এই ডকুমেন্টের ট্রাবলশুটিং সেকশনগুলো তৈরি করা হয়েছে।

---

## অংশ ১: SSH প্রযুক্তির মৌলিক ধারণা

### ১.১ SSH কী এবং কেন প্রয়োজন

#### সংজ্ঞা

SSH (Secure Shell) হলো একটি নেটওয়ার্ক প্রোটোকল যা একটি অনিরাপদ নেটওয়ার্কের উপর দিয়ে দুটি কম্পিউটারের মধ্যে নিরাপদ ডেটা কমিউনিকেশন প্রতিষ্ঠা করে। এটি মূলত রিমোট লগইন, কমান্ড এক্সিকিউশন, এবং ফাইল ট্রান্সফারের জন্য ব্যবহৃত হয়।

#### কেন SSH প্রয়োজন?

**প্রাচীন পদ্ধতির সমস্যা:**
১৯৯০ এর দশকের শুরুতে, সিস্টেম অ্যাডমিনিস্ট্রেটররা রিমোটলি সার্ভার ম্যানেজ করার জন্য Telnet, rlogin, rsh, এবং FTP এর মতো প্রোটোকল ব্যবহার করতেন। এই প্রোটোকলগুলোর একটি বড় সমস্যা ছিল - তারা সব ডেটা প্লেইন টেক্সট আকারে পাঠাতো।

**বাস্তব উদাহরণ:** মনে করুন আপনি ঢাকা থেকে একটি সার্ভার ম্যানেজ করছেন যা চট্টগ্রামে অবস্থিত। আপনি Telnet ব্যবহার করে লগইন করলেন এবং আপনার ইউজারনেম ও পাসওয়ার্ড টাইপ করলেন। এই মুহূর্তে, আপনার নেটওয়ার্ক পথে কেউ যদি প্যাকেট স্নিফিং করে (যেমন: একই নেটওয়ার্কে থাকা কোনো আক্রমণকারী), সে আপনার পাসওয়ার্ড সহজেই দেখতে পাবে। এটি শুধু পাসওয়ার্ডই নয়, আপনি যে সব কমান্ড চালাচ্ছেন, যে সব সংবেদনশীল ডেটা ট্রান্সফার করছেন - সব কিছুই উন্মুক্ত।

**SSH এর সমাধান:**
SSH এই সমস্যার সমাধান করে এনক্রিপশন ব্যবহার করে। SSH কানেকশনের মাধ্যমে পাঠানো সব ডেটা এনক্রিপ্টেড থাকে, ফলে ম্যান-ইন-দ্য-মিডল (Man-in-the-Middle) আক্রমণ থেকে রক্ষা পাওয়া যায়।

#### SSH এর প্রধান ব্যবহারক্ষেত্র

১. **রিমোট সার্ভার ম্যানেজমেন্ট**: 
   - ডেটাসেন্টারে থাকা সার্ভার অফিস থেকে ম্যানেজ করা
   - ক্লাউড ভিএম (Virtual Machine) কনফিগার করা
   - হেডলেস সার্ভার (মনিটর-কিবোর্ড ছাড়া) পরিচালনা

২. **ফাইল ট্রান্সফার**:
   - SFTP (SSH File Transfer Protocol) ব্যবহার করে নিরাপদে ফাইল আদান-প্রদান
   - স্ক্রিপ্টের মাধ্যমে অটোমেটেড ব্যাকআপ
   - ওয়েব সার্ভারে ফাইল আপলোড

৩. **পোর্ট ফরওয়ার্ডিং ও টানেলিং**:
   - নিরাপদে ডাটাবেস কানেকশন
   - ফায়ারওয়ালের আড়ালে থাকা সার্ভিসে এক্সেস
   - VPN এর বিকল্প হিসেবে

৪. **অটোমেশন ও স্ক্রিপ্টিং**:
   - একাধিক সার্ভারে একই সাথে কমান্ড চালানো
   - কনফিগারেশন ম্যানেজমেন্ট টুলস (Ansible, Puppet, Chef)
   - CI/CD পাইপলাইন

### ১.২ SSH এর ইতিহাস ও বিবর্তন

#### প্রথম প্রজন্ম (SSH-1)

১৯৯৫ সালে ফিনিশ কম্পিউটার বিজ্ঞানী Tatu Ylönen SSH প্রোটোকল ডিজাইন করেন। সেই সময় হেলসিঙ্কি ইউনিভার্সিটিতে একটি পাসওয়ার্ড স্নিফিং অ্যাটাকের ঘটনা ঘটে, যা SSH এর জন্ম দেয়।

**SSH-1 এর সীমাবদ্ধতা:**
- ক্রিপ্টোগ্রাফিক দুর্বলতা
- Integrity check এ সমস্যা
- আজকের দিনে এটি obsolete এবং ব্যবহার করা উচিত নয়

#### দ্বিতীয় প্রজন্ম (SSH-2)

২০০৬ সালে SSH-2 প্রোটোকল স্ট্যান্ডার্ড হিসেবে গৃহীত হয় (RFC 4250-4256)।

**SSH-2 এর উন্নতি:**
- শক্তিশালী এনক্রিপশন অ্যালগরিদম
- ভালো key exchange পদ্ধতি
- Multiple sessions over single connection
- Public key authentication এর উন্নত সংস্করণ

#### আধুনিক SSH

বর্তমানে আমরা SSH-2 প্রোটোকলের উন্নত সংস্করণ ব্যবহার করি, যাতে:

- **Ed25519** কী সাপোর্ট (২০১৪ থেকে)
- **ChaCha20-Poly1305** এনক্রিপশন
- **FIDO/U2F** হার্ডওয়্যার কী সাপোর্ট
- **Certificate-based authentication**

### ১.৩ SSH আর্কিটেকচার ও কাজের পদ্ধতি

#### SSH আর্কিটেকচারাল কম্পোনেন্ট

```
┌─────────────────┐         এনক্রিপ্টেড কানেকশন         ┌─────────────────┐
│   SSH Client    │ ◄──────────────────────────────────► │   SSH Server    │
│   (Local PC)    │         TCP Port 22                  │  (Remote Server)│
└─────────────────┘                                       └─────────────────┘
```

**SSH Client:**
- আপনার লোকাল মেশিনে চলে (ল্যাপটপ, ডেস্কটপ)
- সার্ভারের সাথে কানেকশন initiates করে
- কমান্ড পাঠায় এবং আউটপুট গ্রহণ করে
- উদাহরণ: OpenSSH client, PuTTY, SecureCRT

**SSH Server:**
- রিমোট সার্ভারে চলে
- ইনকামিং কানেকশন শোনে (Port 22)
- ক্লায়েন্টকে authenticate করে
- রিকোয়েস্টেড সার্ভিস প্রদান করে
- উদাহরণ: OpenSSH server (sshd)

#### SSH হ্যান্ডশেক প্রক্রিয়া

একটি SSH কানেকশন প্রতিষ্ঠার সময় নিচের ধাপগুলো ঘটে:

**ধাপ ১: TCP কানেকশন**
```
Client → Server: TCP Port 22 এ কানেকশন রিকোয়েস্ট
Server → Client: TCP কানেকশন Accept
```

**ধাপ ২: প্রোটোকল ভার্শন এক্সচেঞ্জ**
```
Server → Client: SSH-2.0-OpenSSH_8.9
Client → Server: SSH-2.0-OpenSSH_8.9
```

**ধাপ ৩: Key Exchange (সবচেয়ে গুরুত্বপূর্ণ)**
- Client এবং Server একত্রিত হয়ে একটি সেশন কী তৈরি করে
- এই কী দিয়ে পরবর্তীতে সব ডেটা এনক্রিপ্ট করা হবে
- Diffie-Hellman key exchange অ্যালগরিদম ব্যবহার করা হয়
- এই ধাপেই সার্ভারের পরিচয় (host key) ভেরিফাই করা হয়

**ধাপ ৪: ইউজার অথেন্টিকেশন**
- সার্ভার ক্লায়েন্টকে authenticate করে
- বিভিন্ন পদ্ধতি হতে পারে:
  - পাসওয়ার্ড
  - পাবলিক কী
  - হোস্ট-বেসড
  - GSSAPI (Kerberos)

**ধাপ ৫: সেশন শুরু**
- অথেন্টিকেশন সফল হলে সেশন শুরু হয়
- কমান্ড এক্সিকিউশন বা ফাইল ট্রান্সফার শুরু হতে পারে

#### এনক্রিপশন প্রক্রিয়া

SSH তিন ধরনের এনক্রিপশন ব্যবহার করে:

১. **Symmetric Encryption (সেশন এনক্রিপশন)**
   - একই কী দিয়ে এনক্রিপ্ট এবং ডিক্রিপ্ট করা হয়
   - খুব দ্রুত কাজ করে
   - উদাহরণ: AES-256-GCM, ChaCha20-Poly1305
   - সেশন শুরু হওয়ার পর সব ডেটা এনক্রিপ্ট করে

২. **Asymmetric Encryption (কী এক্সচেঞ্জ)**
   - Public key এবং private key জোড়া ব্যবহার করে
   - তুলনামূলক ধীরগতির
   - শুধু হ্যান্ডশেকের সময় ব্যবহার হয়
   - উদাহরণ: RSA, ECDSA, Ed25519

৩. **Hashing (Data Integrity)**
   - ডেটার অখণ্ডতা নিশ্চিত করে
   - HMAC (Hash-based Message Authentication Code)
   - উদাহরণ: SHA-256, SHA-512

### ১.৪ SSH Key Types এবং তাদের বৈশিষ্ট্য

SSH কী বিভিন্ন টাইপের হতে পারে। প্রতিটির নিজস্ব সুবিধা ও সীমাবদ্ধতা রয়েছে।

#### RSA (Rivest-Shamir-Adleman)

**বর্ণনা:**
সবচেয়ে পুরনো এবং ব্যাপকভাবে সমর্থিত SSH কী টাইপ। ১৯৭ সালে তৈরি এই অ্যালগরিদম বর্তমানেও সবচেয়ে বেশি ব্যবহৃত হয়।

**প্রযুক্তিগত বিবরণ:**
- Key size: ২৪৮-bit (minimum), ৪০৯৬-bit (recommended)
- Signature algorithm: SHA-256 বা SHA-512
- Compatibility: সব SSH ক্লায়েন্ট ও সার্ভারে চলে

**সুবিধা:**
- সর্বজনীন সাপোর্ট (পুরনো সিস্টেমেও চলে)
- ভালো ডকুমেন্টেশন এবং টুলিং
- FIPS compliant

**অসুবিধা:**
- বড় কী সাইজ (স্লোয়ার পারফরম্যান্স)
- গাণিতিকভাবে তুলনামূলক দুর্বল (quantum computing threat)
- ৩০৭২-bit এর নিচে এখন নিরাপদ নয়

**কখন ব্যবহার করবেন:**
- পুরনো সিস্টেমের সাথে compatibility প্রয়োজন হলে
- FIPS compliance দরকার হলে
- এন্টারপ্রাইজ এনভায়রনমেন্টে

**বাস্তব উদাহরণ:**
একটি ব্যাংকের ডেটাসেন্টারে ১০০+ পুরনো Linux সার্ভার আছে (RHEL 6, CentOS 6)। এই সার্ভারগুলোতে নতুন Ed25519 key সাপোর্ট করে না। সেক্ষেত্রে RSA 4096-bit key ব্যবহার করতে হবে।

#### DSA (Digital Signature Algorithm)

**বর্ণনা:**
১৯৯৪ সালে NIST দ্বারা স্ট্যান্ডার্ডাইজড। OpenSSH 7.0 থেকে DSA keys disable করা হয়েছে।

**সতর্কতা:**
- ❌ **ব্যবহার করবেন না**
- ১০২৪-bit key size fixed (নিরাপদ নয়)
- Modern systems এ সাপোর্ট করে না
- Depreciated

#### ECDSA (Elliptic Curve Digital Signature Algorithm)

**বর্ণনা:**
Elliptic curve cryptography ব্যবহার করে, যা RSA এর চেয়ে ছোট কী সাইজে একই নিরাপত্তা দেয়।

**প্রযুক্তিগত বিবরণ:**
- Key sizes: 256, 384, or 521 bits
- Curve: NIST curves (P-256, P-384, P-521)
- First introduced in OpenSSH 5.7

**সুবিধা:**
- RSA এর চেয়ে ছোট কী (দ্রুত)
- ভালো নিরাপত্তা
- কম CPU usage

**অসুবিধা:**
- NIST curves নিয়ে কিছু নিরাপত্তা উদ্বেগ আছে
- সব পুরনো সিস্টেমে সাপোর্ট করে না
- Implementation জটিল (ভুল হওয়ার সুযোগ)

**কখন ব্যবহার করবেন:**
- Performance critical এনভায়রনমেন্টে
- যখন কী সাইজ গুরুত্বপূর্ণ (embedded systems)
- Modern infrastructure এ

#### Ed25519 (Edwards-curve Digital Signature Algorithm)

**বর্ণনা:**
বর্তমানের সবচেয়ে আধুনিক এবং নিরাপদ SSH key type। 2014 সালে তৈরি এবং OpenSSH 6.5 থেকে সাপোর্ট করে।

**প্রযুক্তিগত বিবরণ:**
- Algorithm: EdDSA (Edwards-curve Digital Signature Algorithm)
- Curve: Curve25519
- Key size: 256-bit (fixed)
- Signature size: 512-bit

**সুবিধা:**
- ⭐ **সবচেয়ে নিরাপদ** (side-channel attack resistant)
- অত্যন্ত দ্রুত (RSA এর চেয়ে ৩-৪ গুণ দ্রুত)
- ছোট কী সাইজ (সহজ ম্যানেজমেন্ট)
- Deterministic signatures (randomness প্রয়োজন নেই)
- সহজ implementation (ভুল হওয়ার সুযোগ কম)

**অসুবিধা:**
- পুরনো সিস্টেমে সাপোর্ট করে না (OpenSSH < 6.5)
- কিছু পুরনো SSH clients এ সমস্যা হতে পারে

**কখন ব্যবহার করবেন:**
- ✅ **সবসময়** (যদি সিস্টেম সাপোর্ট করে)
- নতুন infrastructure সেটআপ
- High security requirements
- Automated systems (দ্রুত authentication)

**বাস্তব উদাহরণ:**
আমাদের Ceph ক্লাস্টারে ৩টি নোড আছে (Ubuntu 24.04)। সব নোডে Ed25519 key ব্যবহার করছি। প্রতিদিন Ansible দিয়ে কনফিগারেশন ম্যানেজমেন্ট করা হয়। Ed25519 key এর কারণে authentication খুব দ্রুত হয়, ফলে ১০০+ সার্ভারে কনফিগারেশন ডেপ্লয় করতে মাত্র ২-৩ মিনিট সময় লাগে। RSA ব্যবহার করলে এই সময় ৮-১০ মিনিট হতো।

#### FIDO/U2F Security Keys

**বর্ণনা:**
হার্ডওয়্যার সিকিউরিটি কী (যেমন: YubiKey, Google Titan) ব্যবহার করে SSH authentication।

**প্রযুক্তিগত বিবরণ:**
- Algorithm: ECDSA বা Ed25519
- Touch required (physical verification)
- PIN protection optional

**সুবিধা:**
- সর্বোচ্চ নিরাপত্তা (physical key প্রয়োজন)
- Phishing resistant
- Multi-factor authentication

**অসুবিধা:**
- Hardware cost
- Key হারালে access হারানোর ঝুঁকি
- সব সিস্টেমে সাপোর্ট করে না

**কখন ব্যবহার করবেন:**
- অত্যন্ত সংবেদনশীল সিস্টেম (প্রোডাকশন, ফাইন্যান্সিয়াল)
- Compliance requirements (SOC2, PCI-DSS)
- Privileged access management

#### Key Type Comparison Table

| বৈশিষ্ট্য | RSA-4096 | ECDSA-256 | Ed25519 | FIDO/U2F |
|-----------|----------|-----------|---------|----------|
| **নিরাপত্তা** | ভালো | খুব ভালো | ⭐ সেরা | ⭐⭐ সর্বোচ্চ |
| **গতি** | ধীর | দ্রুত | অতি দ্রুত | দ্রুত |
| **কী সাইজ** | বড় (7KB) | মাঝারি (600B) | ছোট (100B) | মাঝারি |
| **Compatibility** | সর্বজনীন | আধুনিক | আধুনিক | সীমিত |
| **Quantum Resistant** | ❌ না | ❌ না | ❌ না | ❌ না |
| **Recommended For** | Legacy systems | Performance | Modern systems | High security |

### ১.৫ Authentication Methods

SSH একাধিক authentication method সাপোর্ট করে। প্রতিটির নিজস্ব ব্যবহারক্ষেত্র এবং নিরাপত্তা মাত্রা রয়েছে।

#### Password Authentication

**কাজের পদ্ধতি:**
ইউজার তার পাসওয়ার্ড টাইপ করে, যা এনক্রিপ্টেড হয়ে সার্ভারে পাঠানো হয়। সার্ভার পাসওয়ার্ড ভেরিফাই করে।

**বাস্তব উদাহরণ:**
```
User: ssh root@192.168.1.100
Password: ********
```

**সুবিধা:**
- সহজ সেটআপ (কোনো key management প্রয়োজন নেই)
- সবসময় কাজ করে
- নতুন ইউজারদের জন্য সহজ

**অসুবিধা:**
- ❌ Brute force attack এর ঝুঁকি
- ❌ Weak passwords সহজে crack হয়
- ❌ Automated attacks এ vulnerable
- ❌ Password sharing এর সুযোগ
- ❌ Audit trail দুর্বল

**নিরাপত্তা ঝুঁকি:**
একটি বাস্তব ঘটনা: ২০২৩ সালে একটি স্টার্টআপ কোম্পানির সার্ভারে আক্রমণ হয়। তারা password authentication ব্যবহার করত। আক্রমণকারীরা automated script চালিয়ে ৪৮ ঘণ্টার মধ্যে admin পাসওয়ার্ড brute force করে বের করে ফেলে। পাসওয়ার্ড ছিল "Admin@123" - যা খুবই দুর্বল। ফলে পুরো সার্ভার compromise হয়ে যায়।

**কখন ব্যবহার করবেন:**
- ⚠️ শুধু emergency access এর জন্য
- ⚠️ temporary access (নতুন সার্ভার সেটআপের সময়)
- ❌ কখনোই প্রোডাকশনে একা ব্যবহার করবেন না

#### Public Key Authentication

**কাজের পদ্ধতি:**
এটি asymmetric cryptography ব্যবহার করে। দুটি কী থাকে:
- **Private Key**: আপনার কাছে গোপনে রাখা (কখনো শেয়ার করবেন না)
- **Public Key**: সার্ভারে রাখা (যে কোনো সংখ্যক সার্ভারে শেয়ার করা যাবে)

**Authentication Process:**
```
১. Client সার্ভারে কানেক্ট করে
২. সার্ভার একটি challenge (random data) পাঠায়
৩. Client private key দিয়ে challenge sign করে
৪. সার্ভার public key দিয়ে verify করে
৫. যদি match করে, access দেয়
```

**বাস্তব উদাহরণ:**
```bash
# লোকাল মেশিনে key generate
ssh-keygen -t ed25519 -f ~/.ssh/ceph-key

# Public key সার্ভারে কপি
ssh-copy-id -i ~/.ssh/ceph-key.pub root@192.168.1.100

# এখন password ছাড়া login
ssh -i ~/.ssh/ceph-key root@192.168.1.100
# কোনো password চাইবে না!
```

**সুবিধা:**
- ✅ অত্যন্ত নিরাপদ (private key ছাড়া access অসম্ভব)
- ✅ Brute force resistant
- ✅ Automated scripts এর জন্য perfect
- ✅ Strong audit trail
- ✅ Multi-server management সহজ

**অসুবিধা:**
- Key management overhead
- Private key হারালে access হারানোর ঝুঁকি
- Initial setup একটু জটিল

**কখন ব্যবহার করবেন:**
- ✅ **সবসময়** প্রাথমিক authentication method হিসেবে
- ✅ প্রোডাকশন এনভায়রনমেন্টে
- ✅ Automated systems (CI/CD, Ansible, etc.)
- ✅ Privileged access

#### Keyboard-Interactive Authentication

**কাজের পদ্ধতি:**
এটি একটি flexible authentication method যা multiple challenges support করে। সাধারণত PAM (Pluggable Authentication Modules) এর সাথে কাজ করে।

**ব্যবহারক্ষেত্র:**
- One-Time Password (OTP)
- Two-Factor Authentication (2FA)
- Custom authentication modules

**বাস্তব উদাহরণ (Google Authenticator):**
```bash
User: ssh admin@server
Password: ********
Verification code: 123456  # Google Authenticator app থেকে
```

**সুবিধা:**
- Multi-factor authentication সম্ভব
- Flexible (various methods combine করা যায়)
- Compliance requirements পূরণ করে

**অসুবিধা:**
- জটিল সেটআপ
- User experience একটু কঠিন
- Additional infrastructure প্রয়োজন (OTP server)

#### GSSAPI Authentication (Kerberos)

**কাজের পদ্ধতি:**
Kerberos ticket-based authentication ব্যবহার করে। Enterprise environment এ খুব জনপ্রিয়।

**বাস্তব উদাহরণ:**
```bash
# প্রথমে Kerberos ticket পাওয়া
kinit username@REALM.COM

# তারপর SSH (কোনো password চাইবে না)
ssh user@server.company.com
```

**সুবিধা:**
- Single Sign-On (SSO)
- Centralized user management
- Enterprise-grade security

**অসুবিধা:**
- জটিল infrastructure (Kerberos KDC প্রয়োজন)
- শুধু বড় organization এর জন্য উপযুক্ত
- Maintenance overhead

#### Host-Based Authentication

**কাজের পদ্ধতি:**
Client হোস্টের পরিচয় দিয়ে authentication করা হয়। নির্দিষ্ট হোস্ট থেকে আসা কানেকশন বিশ্বাস করা হয়।

**ব্যবহারক্ষেত্র:**
- Trusted networks
- Internal cluster communication
- Automated system-to-system access

**সুবিধা:**
- User-level management প্রয়োজন নেই
- দ্রুত access

**অসুবিধা:**
- কম নিরাপদ (হোস্ট compromise হলে সব access চলে যায়)
- শুধু বিশেষ ক্ষেত্রে ব্যবহার করা উচিত

#### Authentication Methods Comparison

| Method | নিরাপত্তা | সহজলভ্যতা | Automation | Recommended Use |
|--------|----------|------------|------------|-----------------|
| **Password** | ⭐ | ⭐⭐⭐⭐⭐ | ⭐ | Emergency only |
| **Public Key** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | **Primary method** |
| **Keyboard-Interactive** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | 2FA required |
| **GSSAPI** | ⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐ | Enterprise SSO |
| **Host-Based** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Internal trusted |

#### Multi-Factor Authentication (MFA)

**ধারণা:**
একাধিক authentication method একসাথে ব্যবহার করা। যেমন: Public Key + OTP

**OpenSSH Configuration:**
```bash
# /etc/ssh/sshd_config
AuthenticationMethods publickey,keyboard-interactive
```

**বাস্তব প্রয়োগ:**
একটি ফাইন্যান্সিয়াল প্রতিষ্ঠানে production সার্ভার access এর জন্য MFA বাধ্যতামূলক:
1. প্রথমে private key দিয়ে authenticate করতে হবে
2. তারপর Google Authenticator থেকে OTP দিতে হবে
3. তারপর access পাবে

এটি নিশ্চিত করে যে即使 private key চুরি হয়, আক্রমণকারী access পাবে না কারণ OTP তার কাছে নেই।

---

## অংশ ২: Ceph ক্লাস্টারে Passwordless SSH সেটআপ

### ২.১ প্রয়োজনীয় প্রস্তুতি ও পরিকল্পনা

#### Ceph ক্লাস্টার আর্কিটেকচার বোঝা

Ceph একটি distributed storage system যা একাধিক নোড নিয়ে গঠিত। একটি typical 3-node Ceph ক্লাস্টারে:

```
                    ┌─────────────┐
                    │  Admin Node │
                    │    ceph1    │
                    │ 192.168.68.248 │
                    └──────┬──────┘
                             │
              ┌──────────────┴──────────────┐
              │                              │
       ┌──────▼──────┐              ┌───────▼──────┐
       │  Storage    │              │   Storage    │
       │    Node     │              │     Node     │
       │    ceph2    │              │     ceph3    │
       │192.168.68.249│              │ 192.168.68.250│
       └─────────────┘              └──────────────┘
```

**প্রতিটি নোডের ভূমিকা:**

**Admin Node (ceph1):**
- Ceph ক্লাস্টার ম্যানেজ করে
- Configuration deploy করে
- Monitoring ও maintenance করে
- সব নোডে SSH access প্রয়োজন

**Storage Nodes (ceph2, ceph3):**
- Actual data store করে
- OSD (Object Storage Daemon) চালায়
- MON (Monitor) daemon চালাতে পারে
- Admin node থেকে commands receive করে

#### কেন Passwordless SSH প্রয়োজন?

**বাস্তব পরিস্থিতি:**
মনে করুন আপনার ৩-নোড Ceph ক্লাস্টার আছে। প্রতিদিন আপনাকে:
- ক্লাস্টার স্ট্যাটাস চেক করতে হয়
- Configuration update করতে হয়
- Logs collect করতে হয়
- Backup নিতে হয়
- Monitoring data collect করতে হয়

**Password authentication ব্যবহার করলে:**
```bash
# প্রতিটি command এর জন্য password দিতে হবে
ssh root@ceph1  # password দাও
ceph status     # কাজ করো
exit

ssh root@ceph2  # আবার password দাও
ceph osd df     # কাজ করো
exit

ssh root@ceph3  # আবার password দাও
ceph health     # কাজ করো
exit
```

এটি অত্যন্ত সময়সাপেক্ষ এবং automation এর জন্য অসম্ভব।

**Passwordless SSH ব্যবহার করলে:**
```bash
# কোনো password ছাড়াই সব কাজ
ssh ceph1 "ceph status"
ssh ceph2 "ceph osd df"
ssh ceph3 "ceph health"

# অথবা Ansible দিয়ে একসাথে
ansible ceph-cluster -m command -a "ceph health"
```

**সুবিধা:**
1. **Automation**: Ansible, Puppet, Chef দিয়ে সহজে manage করা যায়
2. **Time Saving**: বারবার password দিতে হয় না
3. **Scripting**: Shell scripts লিখে automated tasks করা যায়
4. **Security**: Strong keys ব্যবহার করা যায় (password এর চেয়ে নিরাপদ)
5. **Audit Trail**: কোন key থেকে access পাওয়া গেছে trace করা যায়

#### প্রয়োজনীয় উপাদান

**হার্ডওয়্যার/সফটওয়্যার Requirements:**

| উপাদান | ন্যূনতম | প্রস্তাবিত |
|--------|--------|-----------|
| **OS** | Ubuntu 20.04 | Ubuntu 24.04 LTS |
| **RAM** | 4 GB | 8 GB+ |
| **CPU** | 2 cores | 4 cores+ |
| **Network** | 1 Gbps | 10 Gbps |
| **SSH Version** | OpenSSH 7.0+ | OpenSSH 8.0+ |

**নেটওয়ার্ক Requirements:**
- সব নোড পরস্পর communicate করতে পারবে
- Port 22 (SSH) open থাকতে হবে
- Static IP addresses ব্যবহার করা ভালো
- Low latency (< 1ms) প্রয়োজন

**সফটওয়্যার প্রস্তুতি:**
```bash
# সব নোডে SSH server install
sudo apt update
sudo apt install openssh-server

# SSH service enable ও start
sudo systemctl enable ssh
sudo systemctl start ssh

# Firewall এ SSH allow
sudo ufw allow ssh
```

#### পরিকল্পনা ও ডকুমেন্টেশন

**IP Address Planning:**
```
Network: 192.168.68.0/24
Gateway: 192.168.68.1

ceph1 (Admin): 192.168.68.248
ceph2 (Storage): 192.168.68.249
ceph3 (Storage): 192.168.68.250
```

**User Planning:**
```
Username: cephuser
Purpose: Ceph cluster management
Sudo access: Yes (passwordless for automation)
Shell: /bin/bash
```

**Key Management Strategy:**
```
Key Type: Ed25519 (modern, fast, secure)
Key Size: 256-bit (fixed for Ed25519)
Passphrase: No (for automation)
Backup: Yes (encrypted backup)
Rotation: Every 6 months
```

### ২.২ পুরনো SSH কনফিগারেশন ক্লিনআপ

#### কেন ক্লিনআপ প্রয়োজন?

**বাস্তব সমস্যা:**
একটি production environment এ আমরা নতুন করে Ceph ক্লাস্টার সেটআপ করছিলাম। পুরনো নোডগুলো আগে অন্য প্রজেক্টে ব্যবহার করা হতো। ক্লিনআপ না করায় নিচের সমস্যাগুলো হয়েছিল:

১. **Conflicting Keys**: পুরনো authorized_keys ফাইলে অনেকগুলো key ছিল। কোন key দিয়ে access পাচ্ছে trace করা কঠিন ছিল।

২. **Wrong Permissions**: পুরনো কনফিগারেশনে কিছু ফাইলের permission ভুল ছিল (777), যা SSH reject করত।

৩. **Known Hosts Conflict**: পুরনো known_hosts ফাইলে IP address গুলো অন্য হোস্টের ছিল। "Host key verification failed" এরর আসত।

৪. **Config Conflicts**: পুরনো ssh_config ফাইলে custom settings ছিল যা নতুন সেটআপের সাথে conflict করত।

**ফলাফল:** ৩ ঘণ্টা debugging করার পর বুঝতে পারি ক্লিনআপ করা প্রয়োজন ছিল।

#### সম্পূর্ণ ক্লিনআপ প্রসিডিউর

**সতর্কতা:** এই ধাপগুলো করার আগে নিশ্চিত হোন যে আপনার Console (VNC) access আছে। SSH access চলে গেলে Console দিয়ে ঢুকতে পারবেন।

**ধাপ ১: বর্তমান SSH কনফিগারেশন ব্যাকআপ**

প্রতিটি নোডে (ceph1, ceph2, ceph3) আলাদাভাবে:

```bash
# ব্যাকআপ ডিরেক্টরি তৈরি
sudo mkdir -p /root/ssh-backup-$(date +%Y%m%d)

# বর্তমান কনফিগারেশন ব্যাকআপ
sudo cp -r /etc/ssh /root/ssh-backup-$(date +%Y%m%d)/
cp -r ~/.ssh /root/ssh-backup-$(date +%Y%m%d)/user-ssh-$(whoami)

# ব্যাকআপ ভেরিফাই
ls -la /root/ssh-backup-$(date +%Y%m%d)/
```

**ধাপ ২: ইউজার লেভেল SSH ফাইল মুছে ফেলা**

```bash
# বর্তমান ইউজারের .ssh ডিরেক্টরি সম্পূর্ণ মুছে ফেলা
rm -rf ~/.ssh

# নিশ্চিত হওয়া
ls -la ~/.ssh
# "No such file or directory" দেখাবে
```

**ধাপ ৩: সিস্টেম লেভেল ক্লিনআপ (ঐচ্ছিক)**

```bash
# SSH ক্লায়েন্ট কনফিগ রিসেট (যদি এডিট করে থাকেন)
sudo mv /etc/ssh/ssh_config /etc/ssh/ssh_config.old
sudo cp /etc/ssh/ssh_config.default /etc/ssh/ssh_config 2>/dev/null || \
  echo "# Default SSH Client Config" | sudo tee /etc/ssh/ssh_config

# SSH হোস্ট কী রিসেট করবেন না!
# এটি করলে সব ক্লায়েন্টের known_hosts এ সমস্যা হবে
```

**ধাপ ৪: Known Hosts ক্লিনআপ**

```bash
# যদি পুরনো নোডের IP পুনরায় ব্যবহার করেন
ssh-keygen -R 192.168.68.248
ssh-keygen -R 192.168.68.249
ssh-keygen -R 192.168.68.250

# অথবা সব known hosts মুছে ফেলা (সতর্কতার সাথে)
rm -f ~/.ssh/known_hosts
rm -f ~/.ssh/known_hosts.old
```

**ধাপ ৫: SSH এজেন্ট ক্লিনআপ**

```bash
# SSH agent থেকে সব key remove
ssh-add -D

# Agent বন্ধ করা (যদি চান)
eval $(ssh-agent -k)
```

#### ক্লিনআপ ভেরিফিকেশন

```bash
# চেক করা যে .ssh ডিরেক্টরি নেই
test -d ~/.ssh && echo "WARNING: .ssh directory exists!" || echo "OK: .ssh removed"

# চেক করা যে কোনো SSH key নেই
ls ~/.ssh/id_* 2>/dev/null && echo "WARNING: Keys still exist!" || echo "OK: No keys found"

# চেক করা যে known_hosts ক্লিন
wc -l ~/.ssh/known_hosts 2>/dev/null || echo "OK: No known_hosts file"
```

#### ক্লিনআপ চেকলিস্ট

- [ ] ব্যাকআপ নেওয়া হয়েছে
- [ ] ~/.ssh ডিরেক্টরি মুছে ফেলা হয়েছে
- [ ] known_hosts ক্লিন করা হয়েছে
- [ ] SSH agent ক্লিন করা হয়েছে
- [ ] Console access আছে (নিশ্চিত হওয়া)
- [ ] ব্যাকআপ ফাইল নিরাপদ স্থানে রাখা হয়েছে

### ২.৩ ইউজার ম্যানেজমেন্ট ও পারমিশন

#### কেন ডেডিকেটেড ইউজার প্রয়োজন?

**বাস্তব উদাহরণ:**
আমরা প্রথমে root ইউজার দিয়ে Ceph ক্লাস্টার ম্যানেজ করতাম। একটি ভুল কমান্ডে production ক্লাস্টারের ৫০% OSD down হয়ে গিয়েছিল। তারপর থেকে আমরা নিচের best practices follow করি:

১. **Root ব্যবহার করি না**: সবসময় regular user ব্যবহার করি, sudo দিয়ে privileged commands চালাই
২. **Audit Trail**: কে কী command চালিয়েছে trace করা যায়
৩. **Principle of Least Privilege**: শুধু প্রয়োজনীয় permission দেই

#### ইউজার তৈরি ও কনফিগারেশন

**ধাপ ১: নতুন ইউজার তৈরি**

প্রতিটি নোডে (ceph1, ceph2, ceph3):

```bash
# নতুন ইউজার তৈরি
sudo adduser cephuser
```

**ইন্টারঅ্যাক্টিভ প্রম্পট:**
```
Adding user `cephuser' ...
Adding new group `cephuser' (1001) ...
Adding new user `cephuser' (1001) with group `cephuser' ...
Creating home directory `/home/cephuser' ...
Copying files from `/etc/skel' ...
New password: **********        # একটি শক্তিশালী পাসওয়ার্ড দিন
Retype new password: ********** # একই পাসওয়ার্ড আবার দিন
passwd: password updated successfully

# নিচের তথ্যগুলো optional - Enter চেপে skip করতে পারেন
Changing the user information for cephuser
Enter the new value, or press ENTER for the default
    Full Name []: Ceph Admin User
    Room Number []: 
    Work Phone []: 
    Home Phone []: 
    Other []: 
Is the information correct? [Y/n] Y
```

**ধাপ ২: Sudo Access দেওয়া**

```bash
# cephuser কে sudo গ্রুপে যুক্ত করা
sudo usermod -aG sudo cephuser

# ভেরিফাই করা
groups cephuser
# Output: cephuser : cephuser sudo
```

**ধাপ ৩: Passwordless Sudo (Automation এর জন্য)**

Automation tools (Ansible, scripts) ব্যবহার করলে passwordless sudo প্রয়োজন:

```bash
# sudoers ফাইল এডিট করা (সতর্কতার সাথে!)
sudo visudo
```

**ফাইলের শেষে যুক্ত করুন:**
```bash
# Ceph user - passwordless sudo for automation
cephuser ALL=(ALL) NOPASSWD:ALL
```

**অথবা শুধু Ceph commands এর জন্য (নিরাপদ):**
```bash
# শুধু Ceph related commands
cephuser ALL=(ALL) NOPASSWD: /usr/bin/ceph, /usr/bin/cephadm, /usr/bin/ceph-volume
cephuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart ceph-*, /usr/bin/systemctl start ceph-*, /usr/bin/systemctl stop ceph-*
```

**সতর্কতা:** `visudo` ব্যবহার করুন, কখনো সরাসরি `/etc/sudoers` এডিট করবেন না। ভুল syntax হলে sudo কাজ করা বন্ধ করে দেবে!

**ধাপ ৪: ইউজার ভেরিফিকেশন**

```bash
# ইউজার তৈরি হয়েছে কিনা চেক
id cephuser
# Output: uid=1001(cephuser) gid=1001(cephuser) groups=1001(cephuser),27(sudo)

# হোম ডিরেক্টরি চেক
ls -ld /home/cephuser
# Output: drwxr-x--- 3 cephuser cephuser 4096 ... /home/cephuser

# Sudo access টেস্ট
su - cephuser
sudo whoami
# Output: root (কোনো password চাইবে না যদি NOPASSWD set করা থাকে)
```

#### ইউজার পারমিশন Best Practices

**ফাইল ও ডিরেক্টরি পারমিশন:**

```bash
# হোম ডিরেক্টরি - অন্যরা যেন পড়তে বা লিখতে না পারে
chmod 750 /home/cephuser
chown cephuser:cephuser /home/cephuser

# .ssh ডিরেক্টরি (পরে তৈরি হবে) - শুধু owner পড়তে/লিখতে পারবে
chmod 700 /home/cephuser/.ssh

# Private key - শুধু owner পড়তে পারবে
chmod 600 /home/cephuser/.ssh/id_ed25519

# Public key - সবাই পড়তে পারবে
chmod 644 /home/cephuser/.ssh/id_ed25519.pub

# authorized_keys - শুধু owner লিখতে পারবে
chmod 600 /home/cephuser/.ssh/authorized_keys
```

**সুবিধা:**
- **নিরাপত্তা**: অন্য ইউজাররা আপনার key দেখতে বা পরিবর্তন করতে পারবে না
- **SSH Requirement**: SSH strict permission check করে, ভুল permission হলে কাজ করবে না
- **Audit**: কে কী করছে trace করা সহজ

#### গ্রুপ ম্যানেজমেন্ট

**Ceph স্পেসিফিক গ্রুপ:**

```bash
# Ceph administration গ্রুপ
sudo groupadd ceph-admin

# cephuser কে গ্রুপে যুক্ত করা
sudo usermod -aG ceph-admin cephuser

# Ceph configuration ফাইলগুলোতে গ্রুপ access দেওয়া
sudo chgrp ceph-admin /etc/ceph/*
sudo chmod 640 /etc/ceph/*
```

### ২.৪ SSH Key জেনারেশন ও ম্যানেজমেন্ট

#### Key জেনারেশন স্ট্র্যাটেজি

**কোথায় জেনারেট করবেন?**

সবসময় **Admin Node (ceph1)** এ key জেনারেট করুন। কারণ:
1. Admin node থেকে সব নোড manage করা হয়
2. Centralized key management সহজ
3. Backup ও rotation সহজ

**কোন Key Type?**

আমাদের ক্ষেত্রে:
- **Ed25519** (প্রাথমিক পছন্দ)
- **RSA 4096** (backup, compatibility এর জন্য)

#### ধাপে ধাপে Key জেনারেশন

**ধাপ ১: Admin Node এ লগইন**

```bash
# ceph1 নোডে cephuser হিসেবে লগইন
ssh cephuser@192.168.68.248

# অথবা Console থেকে
su - cephuser
```

**ধাপ ২: Ed25519 Key জেনারেট**

```bash
ssh-keygen -t ed25519 -C "ceph-cluster-2024" -f ~/.ssh/id_ed25519
```

**প্রম্পট এবং উত্তর:**
```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/cephuser/.ssh/id_ed25519): [Enter চাপুন]
Enter passphrase (empty for no passphrase): [Enter চাপুন - খালি রাখুন]
Enter same passphrase again: [Enter চাপুন - খালি রাখুন]
Your identification has been saved in /home/cephuser/.ssh/id_ed25519
Your public key has been saved in /home/cephuser/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx ceph-cluster-2024
The key's randomart image is:
+--[ED25519 256]--+
|       .o+=*+=.  |
|       . o+o=+   |
|        . .=o.   |
|         o  .    |
|        S .      |
|       . o       |
|      . +        |
|     . o         |
|      .          |
+----[SHA256]-----+
```

**ব্যাখ্যা:**
- `-t ed25519`: Key type Ed25519
- `-C "ceph-cluster-2024"`: Comment (key identify করতে সাহায্য করে)
- `-f ~/.ssh/id_ed25519`: Output file path
- **Passphrase খালি রাখা**: Automation এর জন্য প্রয়োজন (Ansible, scripts)

**সতর্কতা:** Production environment এ passphrase ব্যবহার করা উচিত। সেক্ষেত্রে `ssh-agent` ব্যবহার করে automation করা যায়।

**ধাপ ৩: Key ভেরিফিকেশন**

```bash
# Key files চেক করা
ls -la ~/.ssh/
# Output:
# total 16
# drwx------ 2 cephuser cephuser 4096 Mar 16 10:00 .
# drwxr-x--- 3 cephuser cephuser 4096 Mar 16 09:59 ..
# -rw------- 1 cephuser cephuser  411 Mar 16 10:00 id_ed25519
# -rw-r--r-- 1 cephuser cephuser  103 Mar 16 10:00 id_ed25519.pub

# Public key দেখা
cat ~/.ssh/id_ed25519.pub
# Output: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ceph-cluster-2024

# Key fingerprint চেক করা
ssh-keygen -lf ~/.ssh/id_ed25519
# Output: 256 SHA256:xxxxxxxxxxx ceph-cluster-2024 (ED25519)
```

**ধাপ ৪: পারমিশন সেট করা**

```bash
# .ssh ডিরেক্টরি
chmod 700 ~/.ssh
chown cephuser:cephuser ~/.ssh

# Private key (খুবই গুরুত্বপূর্ণ!)
chmod 600 ~/.ssh/id_ed25519
chown cephuser:cephuser ~/.ssh/id_ed25519

# Public key
chmod 644 ~/.ssh/id_ed25519.pub
chown cephuser:cephuser ~/.ssh/id_ed25519.pub

# ভেরিফাই করা
ls -la ~/.ssh/
```

#### Backup Key জেনারেশন (ঐচ্ছিক কিন্তু recommended)

```bash
# RSA backup key (পুরনো সিস্টেমের compatibility)
ssh-keygen -t rsa -b 4096 -C "ceph-cluster-backup-2024" -f ~/.ssh/id_rsa

# অথবা ECDSA key (performance এর জন্য)
ssh-keygen -t ecdsa -b 521 -C "ceph-cluster-ecdsa-2024" -f ~/.ssh/id_ecdsa
```

#### Key ব্যাকআপ স্ট্র্যাটেজি

**বাস্তব ঘটনা:** একবার একটি কোম্পানির admin এর ল্যাপটপ হারিয়ে গিয়েছিল যেখানে production সার্ভারের private key ছিল। ব্যাকআপ না থাকায় ৫০+ সার্ভারে আলাদা আলাদা করে গিয়ে নতুন key সেটআপ করতে হয়েছিল। ৩ দিন downtime হয়েছিল।

**সঠিক ব্যাকআপ পদ্ধতি:**

```bash
# ১. এনক্রিপ্টেড ব্যাকআপ নেওয়া
cd ~/.ssh
tar czf - id_ed25519 id_ed25519.pub | \
  openssl enc -aes-256-cbc -salt -out ceph-key-backup-$(date +%Y%m%d).enc

# ২. নিরাপদ স্থানে রাখা
# - Encrypted USB drive
# - Password manager (যেমন: KeePass, 1Password)
# - Secure cloud storage (encrypted)

# ৩. ব্যাকআপ ভেরিফাই করা
openssl enc -aes-256-cbc -d -in ceph-key-backup-20240316.enc | \
  tar xzf - -C /tmp/key-test/
diff ~/.ssh/id_ed25519 /tmp/key-test/id_ed25519
# কোনো output না আসলে backup ঠিক আছে
```

### ২.৫ Key Distribution পদ্ধতি

#### ssh-copy-id ব্যবহার করে (সহজ পদ্ধতি)

**ধাপ ১: নিজের নোডে (ceph1) Key যুক্ত করা**

```bash
# ceph1 থেকে run করুন
ssh-copy-id -i ~/.ssh/id_ed25519.pub cephuser@192.168.68.248
```

**Output:**
```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/cephuser/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
cephuser@192.168.68.248's password:  # একবার password দিন

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'cephuser@192.168.68.248'"
and check to make sure that only the key(s) you wanted were added.
```

**ধাপ ২: দ্বিতীয় নোডে (ceph2) Key যুক্ত করা**

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub cephuser@192.168.68.249
```

**ধাপ ৩: তৃতীয় নোডে (ceph3) Key যুক্ত করা**

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub cephuser@192.168.68.250
```

**সুবিধা:**
- ✅ সহজ এবং দ্রুত
- ✅ Automatic permission set করে
- ✅ Duplicate key যোগ করে না
- ✅ সঠিক format এ যোগ করে

#### ম্যানুয়ালি Key Distribution (যদি ssh-copy-id কাজ না করে)

**বাস্তব সমস্যা:** কিছু ক্ষেত্রে ssh-copy-id কাজ করে না:
1. পুরনো SSH version
2. Custom SSH configuration
3. Network issues

**ম্যানুয়াল পদ্ধতি:**

**ধাপ ১: Public key কপি করা**

```bash
# ceph1 থেকে public key দেখা
cat ~/.ssh/id_ed25519.pub

# Output copy করুন (পুরো লাইন):
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ceph-cluster-2024
```

**ধাপ ২: প্রতিটি নোডে পেস্ট করা**

প্রতিটি নোডে (ceph1, ceph2, ceph3) Console বা password SSH দিয়ে লগইন করে:

```bash
# .ssh ডিরেক্টরি তৈরি (যদি না থাকে)
mkdir -p ~/.ssh

# authorized_keys ফাইল তৈরি/এডিট
nano ~/.ssh/authorized_keys

# Copy করা public key পেস্ট করুন
# (Ctrl+O দিয়ে save, Ctrl+X দিয়ে exit)

# পারমিশন সেট করা
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R cephuser:cephuser ~/.ssh
```

**ধাপ ৩: ভেরিফিকেশন**

```bash
# authorized_keys এ key আছে কিনা চেক
cat ~/.ssh/authorized_keys

# পারমিশন চেক
ls -la ~/.ssh/
# authorized_keys -rw------- হতে হবে
```

#### একাধিক Key যুক্ত করা

একাধিক admin বা system এর key যুক্ত করতে:

```bash
# Admin 1 এর key
ssh-ed25519 AAAA... admin1@company.com

# Admin 2 এর key
ssh-ed25519 AAAA... admin2@company.com

# Ansible server এর key
ssh-ed25519 AAAA... ansible@automation.local

# Backup key
ssh-rsa AAAA... backup-key-2024
```

**প্রতিটি key আলাদা লাইনে** থাকতে হবে।

#### Mutual Trust Setup (All-to-All SSH)

**প্রয়োজন:** যদি ceph2 থেকে ceph3 এ passwordless SSH প্রয়োজন হয়

**পদ্ধতি ১: একই Private Key সব নোডে (Lab/Testing এর জন্য)**

⚠️ **সতর্কতা:** Production এ এই পদ্ধতি ব্যবহার করবেন না!

```bash
# ceph1 থেকে private key copy করা
cat ~/.ssh/id_ed25519
# পুরো key copy করুন

# ceph2 তে লগইন করে
nano ~/.ssh/id_ed25519
# paste করুন
chmod 600 ~/.ssh/id_ed25519

# ceph3 তে একই কাজ
```

**পদ্ধতি ২: প্রতিটি নোডের আলাদা Key (Production এর জন্য)**

```bash
# ceph2 তে নতুন key generate
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "ceph2-node-key"

# ceph2 এর public key ceph1 ও ceph3 তে যুক্ত করুন
ssh-copy-id -i ~/.ssh/id_ed25519.pub cephuser@ceph1
ssh-copy-id -i ~/.ssh/id_ed25519.pub cephuser@ceph3

# একই কাজ ceph3 তে করুন
```

### ২.৬ SSH Config ফাইল কনফিগারেশন

#### কেন SSH Config প্রয়োজন?

**সমস্যা:** প্রতিবার পুরো command টাইপ করতে হয়:
```bash
ssh cephuser@192.168.68.248 -i ~/.ssh/id_ed25519 -o StrictHostKeyChecking=no
ssh cephuser@192.168.68.249 -i ~/.ssh/id_ed25519 -o StrictHostKeyChecking=no
ssh cephuser@192.168.68.250 -i ~/.ssh/id_ed25519 -o StrictHostKeyChecking=no
```

**সমাধান:** SSH config ফাইল ব্যবহার করে সহজ করা:
```bash
ssh ceph1
ssh ceph2
ssh ceph3
```

#### SSH Config ফাইল তৈরি

**ধাপ ১: ফাইল তৈরি**

```bash
# ceph1 (Admin node) এ
nano ~/.ssh/config
```

**ধাপ ২: কনফিগারেশন লেখা**

```bash
# Ceph Cluster SSH Configuration
# Created: 2024-03-16
# Author: Sumon

# Default settings for all hosts
Host *
    User cephuser
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    AddKeysToAgent yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
    TCPKeepAlive yes
    ConnectionAttempts 3
    ConnectTimeout 10

# Admin Node
Host ceph1
    HostName 192.168.68.248
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel QUIET

# Storage Node 1
Host ceph2
    HostName 192.168.68.249
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel QUIET

# Storage Node 2
Host ceph3
    HostName 192.168.68.250
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel QUIET

# Quick access to all nodes (for Ansible/Scripts)
Host ceph-all
    HostName 192.168.68.248
    # This is just an alias, use ceph1, ceph2, ceph3 individually
```

**ধাপ ৩: পারমিশন সেট করা**

```bash
chmod 600 ~/.ssh/config
chown cephuser:cephuser ~/.ssh/config
```

#### Configuration Options ব্যাখ্যা

**Global Options (Host *):**

| Option | Value | ব্যাখ্যা |
|--------|-------|----------|
| `User` | cephuser | সবসময় এই user দিয়ে login করবে |
| `IdentityFile` | ~/.ssh/id_ed25519 | কোন private key ব্যবহার করবে |
| `IdentitiesOnly` | yes | শুধু specified key ব্যবহার করবে (অন্য key ignore করবে) |
| `AddKeysToAgent` | yes | key ssh-agent এ add করবে (passphrase থাকলে সুবিধা) |
| `ServerAliveInterval` | 60 | ৬ সেকেন্ড পর পর keepalive পাঠাবে (connection drop রোধ) |
| `ServerAliveCountMax` | 3 | ৩ বার failed হলে connection বন্ধ করবে |
| `TCPKeepAlive` | yes | TCP level keepalive |
| `ConnectionAttempts` | 3 | ৩ বার connect চেষ্টা করবে |
| `ConnectTimeout` | 10 | ১০ সেকেন্ড timeout |

**Host-Specific Options:**

| Option | Value | ব্যাখ্যা |
|--------|-------|----------|
| `HostName` | IP address | প্রকৃত server address |
| `StrictHostKeyChecking` | no | Host key verify করবে না (lab এর জন্য) |
| `UserKnownHostsFile` | /dev/null | known_hosts এ save করবে না |
| `LogLevel` | QUIET | কম log দেখাবে |

**সতর্কতা:** Production এ `StrictHostKeyChecking no` ব্যবহার করবেন না! এটি Man-in-the-Middle attack এর ঝুঁকি বাড়ায়।

#### Advanced Config Examples

**Wildcard Hosts:**
```bash
# সব ceph নোডের জন্য একই config
Host ceph*
    User cephuser
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
```

**Multiple Identity Files:**
```bash
Host ceph1
    HostName 192.168.68.248
    IdentityFile ~/.ssh/id_ed25519
    IdentityFile ~/.ssh/id_rsa
```

**Port Forwarding:**
```bash
Host ceph1-db
    HostName 192.168.68.248
    LocalForward 5432 localhost:5432  # PostgreSQL port forward
```

**Proxy Jump (Bastion Host):**
```bash
# Bastion host (public)
Host bastion
    HostName 203.0.113.10
    User admin

# Internal node (via bastion)
Host ceph-internal
    HostName 10.0.0.100
    User cephuser
    ProxyJump bastion
```

### ২.৭ পারমিশন ম্যানেজমেন্ট

#### কেন পারমিশন গুরুত্বপূর্ণ?

**বাস্তব সমস্যা:** একবার একটি production server এ SSH কাজ করা বন্ধ করে দিয়েছিল। ২ ঘণ্টা debugging এর পর দেখা গেল:
```bash
ls -la ~/.ssh/authorized_keys
# Output: -rw-rw-rw- 1 user user 400 ...
```
Permission 666 (সবাই পড়তে ও লিখতে পারে) ছিল! SSH security এর জন্য এটি গ্রহণযোগ্য নয়।

**SSH এর Strict Permission Check:**
SSH খুব strict about permissions। ভুল permission হলে:
- Private key ignore করে
- Connection reject করে
- "Permission denied (publickey)" error দেয়

#### সঠিক পারমিশন সেটিংস

**ডিরেক্টরি পারমিশন:**

```bash
# হোম ডিরেক্টরি
chmod 750 /home/cephuser
# অথবা (আরও strict)
chmod 700 /home/cephuser

# .ssh ডিরেক্টরি
chmod 700 ~/.ssh
# Owner: read, write, execute
# Group: nothing
# Others: nothing
```

**ফাইল পারমিশন:**

```bash
# Private key (সবচেয়ে গুরুত্বপূর্ণ!)
chmod 600 ~/.ssh/id_ed25519
# Owner: read, write
# Group: nothing
# Others: nothing

# Public key
chmod 644 ~/.ssh/id_ed25519.pub
# Owner: read, write
# Group: read
# Others: read

# authorized_keys
chmod 600 ~/.ssh/authorized_keys
# Owner: read, write
# Group: nothing
# Others: nothing

# SSH config
chmod 600 ~/.ssh/config
# Owner: read, write
# Group: nothing
# Others: nothing

# known_hosts
chmod 644 ~/.ssh/known_hosts
# Owner: read, write
# Group: read
# Others: read
```

**Ownership:**

```bash
# সব ফাইল ও ডিরেক্টরির owner সঠিক কিনা চেক
chown -R cephuser:cephuser ~/.ssh/
```

#### পারমিশন ভেরিফিকেশন স্ক্রিপ্ট

```bash
#!/bin/bash
# SSH Permission Checker

echo "Checking SSH permissions for $(whoami)..."

# Check .ssh directory
if [ -d ~/.ssh ]; then
    perms=$(stat -c %a ~/.ssh)
    if [ "$perms" = "700" ]; then
        echo "✓ ~/.ssh permissions: $perms (OK)"
    else
        echo "✗ ~/.ssh permissions: $perms (Should be 700)"
    fi
else
    echo "! ~/.ssh directory not found"
fi

# Check private keys
for key in ~/.ssh/id_ed25519 ~/.ssh/id_rsa ~/.ssh/id_ecdsa; do
    if [ -f "$key" ]; then
        perms=$(stat -c %a "$key")
        if [ "$perms" = "600" ]; then
            echo "✓ $(basename $key) permissions: $perms (OK)"
        else
            echo "✗ $(basename $key) permissions: $perms (Should be 600)"
        fi
    fi
done

# Check authorized_keys
if [ -f ~/.ssh/authorized_keys ]; then
    perms=$(stat -c %a ~/.ssh/authorized_keys)
    if [ "$perms" = "600" ]; then
        echo "✓ authorized_keys permissions: $perms (OK)"
    else
        echo "✗ authorized_keys permissions: $perms (Should be 600)"
    fi
fi

# Check ownership
owner=$(stat -c %U ~/.ssh)
if [ "$owner" = "$(whoami)" ]; then
    echo "✓ Ownership: $owner (OK)"
else
    echo "✗ Ownership: $owner (Should be $(whoami))"
fi
```

#### সাধারণ পারমিশন সমস্যা ও সমাধান

**সমস্যা ১: Private key permission too open**
```bash
Error: Permissions 0644 for '/home/user/.ssh/id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
```

**সমাধান:**
```bash
chmod 600 ~/.ssh/id_rsa
```

**সমস্যা ২: ডিরেক্টরি writable by group**
```bash
Error: Authentication refused: bad ownership or modes for directory /home/user
```

**সমাধান:**
```bash
chmod 755 /home/user
# অথবা
chmod 750 /home/user
```

**সমস্যা ৩: authorized_keys writable by others**
```bash
Error: Authentication refused: bad ownership or modes for file ~/.ssh/authorized_keys
```

**সমাধান:**
```bash
chmod 600 ~/.ssh/authorized_keys
chown user:user ~/.ssh/authorized_keys
```

### ২.৮ ভেরিফিকেশন ও টেস্টিং

#### বেসিক কানেক্টিভিটি টেস্ট

**ধাপ ১: প্রতিটি নোডে আলাদাভাবে টেস্ট**

```bash
# ceph1 থেকে
ssh ceph1 hostname
# Expected output: ceph1

ssh ceph2 hostname
# Expected output: ceph2

ssh ceph3 hostname
# Expected output: ceph3
```

**ধাপ ২: Verbose mode এ টেস্ট (সমস্যা হলে)**

```bash
ssh -v ceph2 hostname
```

**Output দেখে বুঝবেন:**
```
debug1: Connecting to 192.168.68.249 [192.168.68.249] port 22.
debug1: Connection established.
debug1: identity file /home/cephuser/.ssh/id_ed25519 type 3
debug1: Authentications that can continue: publickey,password
debug1: Next authentication method: publickey
debug1: Will attempt key: /home/cephuser/.ssh/id_ed25519 ED25519 SHA256:xxx
debug1: Authentication succeeded (publickey).
debug1: channel 0: new [client-session]
debug1: Sending command: hostname
```

**সফল authentication:**
- "Authentication succeeded" দেখাবে
- কোনো password চাইবে না
- hostname output আসবে

**ব্যর্থ authentication:**
- "Permission denied (publickey)" দেখাবে
- বারবার password চাইবে
- Connection closed হবে

#### অটোমেটেড টেস্টিং

**সব নোডে একসাথে টেস্ট:**

```bash
#!/bin/bash
# Test SSH connectivity to all nodes

nodes=("ceph1" "ceph2" "ceph3")

for node in "${nodes[@]}"; do
    echo "Testing $node..."
    if ssh -o ConnectTimeout=5 -o BatchMode=yes "$node" "echo 'Success: $(hostname)'"; then
        echo "✓ $node is reachable"
    else
        echo "✗ $node connection failed"
    fi
    echo "---"
done
```

**BatchMode=yes** ব্যবহারের সুবিধা:
- Password চাইলে automatically fail করে
- Script hang করে না
- Automation friendly

#### পারমিশন ও কনফিগারেশন ভেরিফিকেশন

**ধাপ ১: সব নোডে চেক**

```bash
# প্রতিটি নোডে লগইন করে
for node in ceph1 ceph2 ceph3; do
    echo "=== Checking $node ==="
    ssh "$node" "
        echo 'User: \$(whoami)'
        echo 'Home: \$HOME'
        echo '.ssh permissions: \$(stat -c %a ~/.ssh)'
        echo 'Private key permissions: \$(stat -c %a ~/.ssh/id_ed25519 2>/dev/null || echo "N/A")'
        echo 'authorized_keys permissions: \$(stat -c %a ~/.ssh/authorized_keys 2>/dev/null || echo "N/A")'
        echo 'Sudo access: \$(sudo -n whoami 2>/dev/null || echo "No passwordless sudo")'
    "
    echo ""
done
```

#### পারফরম্যান্স টেস্টিং

**Connection time মাপা:**

```bash
# সময় নিয়ে SSH connect
time ssh ceph2 "echo 'Connection successful'"

# Output:
# Connection successful
# real    0m0.234s
# user    0m0.123s
# sys     0m0.045s
```

**একাধিক connection একসাথে:**

```bash
# ১০টি parallel connection
time for i in {1..10}; do
    ssh ceph2 "echo 'Connection $i'" &
done
wait
```

#### Troubleshooting টেস্ট

**যদি connection fail করে:**

```bash
# ১. Network connectivity চেক
ping -c 3 192.168.68.249

# ২. Port 22 open আছে কিনা চেক
nc -zv 192.168.68.249 22
# অথবা
telnet 192.168.68.249 22

# ৩. SSH service চলছে কিনা চেক
ssh ceph2 "systemctl status sshd"

# ৪. Firewall চেক
ssh ceph2 "sudo ufw status"

# ৫. SSH logs চেক
ssh ceph2 "sudo tail -50 /var/log/auth.log | grep sshd"

# ৬. Very verbose mode
ssh -vvv ceph2 hostname
```

#### Final Validation Checklist

সব কাজ শেষে এই checklist follow করুন:

- [ ] সব নোডে passwordless SSH কাজ করছে
- [ ] `ssh ceph1`, `ssh ceph2`, `ssh ceph3` - সব কাজ করছে
- [ ] কোনো password চাইছে না
- [ ] Sudo access কাজ করছে (passwordless)
- [ ] পারমিশন সব জায়গায় সঠিক (700, 600, 644)
- [ ] SSH config ফাইল সঠিকভাবে কাজ করছে
- [ ] Backup নেওয়া হয়েছে
- [ ] Documentation আপডেট করা হয়েছে

---

**নোট:** এই ডকুমেন্টের পরবর্তী অংশগুলোতে SSH কনফিগারেশন ট্রাবলশুটিং, OpenStack ইনস্ট্যান্স রিকভারি, নিরাপত্তা best practices, এবং অপারেশনাল গাইডলাইন বিস্তারিত আলোচনা করা হবে। ডকুমেন্টের দৈর্ঘ্যের কারণে এটি একাধিক অংশে বিভক্ত।

(ডকুমেন্ট চলমান...)
