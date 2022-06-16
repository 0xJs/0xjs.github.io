---
future: true
published: true


title: "PNPT review"
date: 2022-06-16 16:51:00 +0200
categories: [Ethical Hacking, Reviews]
tags: [ethical hacking]     # TAG names should always be lowercase
author:
  name: "Jony Schats"
---

## Introduction
I just passed the [Practical Network Penetration Tester](https://certifications.tcm-sec.com/pnpt/) exam. I heard a lot of good things about the course and I also received a lot of questions about if I knew the course was good. So, I decided to take the course and exam so I knew where the course was about and how good it was.

## The material
The PNPT course consists out of five modules which can be bought as standalone courses, consisting off:

- Practical Ethical Hacking
- Linux Privilege Escalation for Beginners
- Windows Privilege Escalation for Beginners
- Open Source Intelligence (OSINT) Fundamentals
- External Pentest Playbook

The courses covers everything needed to be able to perform a pentest on a company. From OSINT to external and internal networks, including basic Active Directory attacks and privilege escalation on Windows and Linux.

I didn't spend too much time within the course since I already had a lot of knowledge in pentesting Networks and Active Directory, so I skipped through and watched all videos at twice the speed. The whole material is purely video based with slides in the background where the cybermentor is talking about the topics in the slides. Overall the videos and the content were good, way better than some of the most recognisable certifications.

One thing I was missing since I wanted to go through the material quickly was a slide deck for each section.

### The labs
The course doesn't have a dedicated lab included and you need to download your own machines and run them in a hypervisor. Everything needed to build this is covered in the material, including building your own small Active Directory environment. This could be seen as a bad thing, but you will learn a lot by building your own lab and it will learn you the basics and it gives you the ability to build upon it.

## The exam
When you book the exam, you won't get anything in advance. At the time the exam starts you will receive your VPN pack and the rules of engagement document describing the engagement and defining the scope. The exam is a simulation of a real-world penetration test / vulnerability assessment where you need to report back to a fake customer. You will need to perform Open-Source Intelligence, hack into the external network and move laterally into the internal network and compromise the Domain Controller to complete the exam lab.

As said the exam lab was setup like a real engagement would be and I really liked this type of exam. It is a good simulation of how a real penetration test might be.

When the exam starts you are given five full days to hack into all the systems and an additional two days to write the report. For a beginner this should be plenty of time to finish the exam. The time given for the report writing could be a bit longer because it might be tough for beginners writing their first pentest report. I remember that it took me two full days to write my EWPT report for eLearnSecurity since it was my first ever penetration testing report and it took me quite some time. I would definitely recommend creating your own template in advance or using the one from [TCM Security](https://github.com/hmaverickadams/TCM-Security-Sample-Pentest-Report) and definitely going through their example report.

To pass the exam a student need to:

- Compromise the external and internal network
- Write a professional report
- Perform a live 15 minute debrief / presentation

### My exam experiences
I started the exam at 9:30 in the morning and received my VPN pack a bit later. It took me 3 hours to complete my OSINT and get my initial access on the first machine. For some reason I always stress the first couple of hours of an exam and because of it I made an oopsie in the OSINT, which led me to not being able to get my initial access. After taking a small break and going back to the OSINT I saw my mistake, a typo -__- and did my attacks again and got access to the first machine.

After gaining access to the first machine I owned one after the other and gained access to multiple machines within an hour. Getting to the DC took me the longest since I was looking for complicated AD stuff that wasn't taught in the course. I took a break and did some basic enumeration and found the vector which led me to Domain Admin privileges. 

I completed the exam in 8 hours and started going through my notes and screenshots to double check if I had everything. After that I did some more enumeration in the environment and looked for other vulnerabilities to report. The exam is not about just owning all machines but about finding vulnerabilities to report to the ''customer''.

After I was sure I got all my notes I got a good night of sleep and started writing the report. The writing of the report took me about 4-5 hours and I read through it again a couple hours later and the next day to turn it in. My report existed of 34 pages. After a couple of hours, I got the e-mail that I passed the reporting fase and that I could book a time for the debrief. Booking the time for the debrief was possible for the next day already, but since I didn't have time that day I booked it two days later.

I created a simple presentation for the debrief containing a summary of the different vulnerabilities found and explained the path of exploitation which lead to the compromise of the DC.

## Tips for the certifiation
- Prepare a template report and go through TCM security example report
- Check out my cheatsheets:
  - Infrastructure: https://github.com/0xJs/RedTeaming_CheatSheet/tree/main/infrastructure
  - Active Directory (CRTP) simplified version of the AD part in my redteaming cheatsheet: https://github.com/0xJs/CRTP-cheatsheet
- Take your time during the exam to sleep properly, take breaks, go for walks etc. Especially having a walk outside can really help when being stuck by going over what you have tried etc.

## Conclusion
I really liked the setup of the course and the exam environment. It simulates a real-life scenario penetration test and learns you the basics on how to perform one. It is the perfect course for beginners and I wish I was able to take it when I started my Ethical Hacking journey two years ago.
