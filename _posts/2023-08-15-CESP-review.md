---
future: true
published: true


title: "CESP review"
date: 2023-08-15 10:00:00 +0200
categories: [Ethical Hacking, Reviews]
tags: [ethical hacking, red teaming]     # TAG names should always be lowercase
author:
  name: "Jony Schats"
---

## Introduction
Altered Security released the new course [Certified Enterprise Security Professional – AD CS (CESP)](https://www.alteredsecurity.com/adcs). This course is pureply focussed on Active Directory Certificate Services (ADCS). It is required to have basic Active Directory(AD) knowledge before starting the course, basicly CRTP level. I have a decent amount of knowledge about AD, but I never played with ADCS and my knowledge was limited on it. I never fully read the famous ADCS [Certified Pre-Owned](https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf) whitepaper by SpecterOps. 

The course basiscly covers everything from the paper that still works till this day since Microsoft patched a couple of attacks and it covers some extra certificate stuff.

## The material & course

The course focusses on ADCS and will teach you everything about;

- how to escalate privileges locally using CertPotato
- how to escalate privileges abusing certificate templates - ESCx attacks
- how to steal certificates - THEFTx attacks
- how to build persistence with certificates - (D)PERSTISTx attacks
- how to sign your own executables with stolen signing certificates bypassing WDAC policies
- how to forge a certificate to authenticate to Azure using a stolen Certificate Authority

The course with exam starts at 250 euros for a month access. A full curriculum can be found [here, click on What will you Learn?](https://www.alteredsecurity.com/adcs). All the new thing I learned will be added to my [RedTeaming CheatSheet](https://github.com/0xJs/RedTeaming_CheatSheet) soon.

## The labs
The labs and the material go hand in hand and will guide you through the attack paths inside the lab. The lab exists out of three forests and an Azure tenant.

The lab is from the ''Assume Breach'' perspective which means you are given valid AD credentials and access to a machine to perform the attacks from. All tools required are already installed on the machine and there is also WSL to perform the attacks from the Linux perspective.

There are two options to connect to the lab. One is using the VPN pack provided and RDP into the machine and the other is using Apache Guacamole web access to the machine. The first option was very stable for me, I didn't try the web access since I dislike Apache Guacamole. The machines in the run 24/7 and it is a shared environment with other students. This has it benefits such as you can keep Terminals open and keep continuing after leaving the pc for a while or after reconnecting the next day. But from time to time you mind stumble upon another student their tools on a machine.

The lab for me was very stable, the material and lab manual are very good and can be followed easily. The total video material is a bit longer then 11 hours with 223 slides. The lab manual is 206 pages long. The lab and material is categorized in objectives(exercises) which the student should perform. In the lab portal there are two questions(flags/info to be collected) which should be answered for each objective.

The only thing I disliked about the course is the way the video material is categorized since it isn't fully in sync with the slides. The video's aren't ordered correctly, the video about how to complete the objective using Linux or Windows is before the video which explains the theory behind the attack. Which got me confused quite a bit sometimes.

## The exam
The exam environment exists out of 4 machines excluding the starting machine and the goal of the exam is to get code execution on all of em. Once the exam starts you are given 25 hours to compromise the environment and then 48 hours to write the report. The exam environment has the same connectivity options as in the lab, either VPN or Apache Guacamole via the browser.

## My exam experience
I started the exam around 9:30 AM and had access to all machines around 12:30 AM, so there is plenty of time for the exam. I went through it quickly but it would take someone a bit more time I would guess. The exam is quite easy, just above CRTP level and doesn't contain new techniques not taught in the course.

## Conclusion
I really liked the course and the lab. It is another great course teached by Nikhil from Altered Security.