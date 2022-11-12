---
future: true
published: true


title: "Global Central Bank (GCB) review"
date: 2022-06-16 16:51:00 +0200
categories: [Ethical Hacking, Reviews]
tags: [ethical hacking]     # TAG names should always be lowercase
author:
  name: "Jony Schats"
---

## Introduction
I just passed the [PentesterAcademy Certified Enterprise Security Specialist](https://www.pentesteracademy.com/gcb) exam. The lab and the course is made by [PentesterAcademy](https://www.pentesteracademy.com) and is known as the Global Central Bank. It is the biggest red teaming Active Directory lab they they offer.

## The material
The GCB course consists out of nine videos with a total length of around 3 hours. The video's covers topics such as:

- PAM Trusts
- Local Administrator Password Solution (LAPS)
- PowerShell Web Access (PSWA)
- Windows Linux subsystem (WSL)
- Unconstrained delegation and the printer bug
- Resource Based Constrained Delegation (RBCD)
- Just Enough Admin (JEA)
- Exchange

The supplements on the already thought material from CRTP and CRTE and its expected that you know the material from those courses and know a decent amount about On-Prem Active Directory and AD exploitation & misconfigurations. With the material they also give some attack path diagrams for whenever you are stuck which gives small hints on what the next machine should be.

### The labs
The lab consists out of 7 forests, 9 domains and around 25-30 machines. The lab exercises are split into 12 sections and multiple attack paths. When you start the lab you are provided with a VPN pack and Apache Guacamole browser access. I prefered the lab pack and used that to RDP into the Windows attacking machine. It is expected that you execute the attacks from this VM and bring your own toolset inside the lab. There is nothing installed on the machine and you don't start with high privileges. 

The lab was big and is good designed which made it fun to exploit everything and progress in the lab. I was already familiar with most abuses and misconfigurations except PAM, PSWA, WSL and JEA. The part were I struggled the most was in the post-exploitation fase where you needed to find some kind of information from enterprise applications or for example the DNS service on the DC. However whenever you are stuck you can mail the lab team and they will give you some information on how to progress further! The lab really looks like the drawing on their website!


## The exam
The exam environment exists out of five machines with the goal to get command execution on each of them. A big difference compared to CRTP and CRTE is that it is also required to fully remediate the misconfigurations and abuses while not breaking the lab. They will give a list of functionalities which should still work to pass the exam. It is also required to fully document the implementation of the remediation and why you chosen to do so.

### My exam experiences


## Tips for the certifiation


## Conclusion

