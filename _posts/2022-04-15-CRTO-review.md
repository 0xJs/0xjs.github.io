---
future: true
published: true


title: "CRTO review"
date: 2022-04-17 20:57:00 +0200
categories: [Ethical Hacking, Reviews]
tags: [ethical hacking, red teaming]     # TAG names should always be lowercase
author: jony
---

## Introduction
Last week I passed the [Certified Red Team Operator](https://courses.zeropointsecurity.co.uk/courses/red-team-ops) (CRTO) exam. I have taken multiple courses about pentesting Active Directory (AD), this is the 6th lab and the 4th certification. The Active Directory part in the course is not very extensive, but the personal labs and overall experience were good. They weren't slow or unstable like in eCPTX. The course mostly focusses on Red Teaming in a mostly Windows environment.

You will get a personal lab with 40 hours of access which will only tick down the time the labs are running (you can freely stop and start them). It is recommended to take the course if you already have a basic understanding of Active Directory and its misconfigurations. By for example completing CRTP.

## The material

The course has a focus on Red Teaming and will teach you about topics such as:

- What is red teaming and OPSEC
- Command and Control (The course teaches Cobalt Strike)
- Initial/host/domain reconnaissance
- Lateral movement
- Privilege escalation
- Pivoting
- Basic Active Directory abuses

The course with exam costs around 400 euros. A full curriculum can be found [here](https://courses.zeropointsecurity.co.uk/courses/red-team-ops). All the new thing I learned are added to my [RedTeaming CheatSheet](https://github.com/0xJs/RedTeaming_CheatSheet) and I also made a [Cobalt Strike page](https://github.com/0xJs/RedTeaming_CheatSheet/blob/main/cobalt-strike.md).

## The labs
The labs and the material go hand in hand and will guide you through the labs and the attack paths. You start with 40 hours of lab access and can always buy more time. I only used 20 hours of it because I tried to save as much time as possible by reading all the material and taking all my notes before diving into the lab. But this wasn't neccesary since you have plenty of time to finish the lab. 

The lab didn't teach me many new Active Directory concepts (I did CRTP, CRTE and eCPTX) but it did teach me how to do some attacks in different ways. One of the recently added things in the lab is Active Directory Certificate Services, which was new for me too. I also learned to be familiair with creating and using tickets instead of using Pass The Hash attacks for everything. Besides that I learned a lot about using Cobalt Strike and using the pivoting functionaility is amazing. It is way easier then doing everything manually! You can do one command and tunnel through multiple hops back to your teamserver, and even setup proxychains through all your agents! Its awesome!

The lab also has a Kibana instance which collects all the logs where you would be able to detect your own attacks and create your own rules, but I didn't play to much with it (yet, I will do this with the 20 hours left). The course inclused examples on how to detect all of the attacks.

The lab is closed off and you won't get a VPN to connect to the lab with our own machine. You will have to use the attacker kali and windows machine in the lab which you can access through Guacamole. You will be able to access any machine in the lab through it in case you need to troubleshoot something. There is a CRTO discord on the contact [page](https://www.zeropointsecurity.co.uk/contact)

## The exam
When you book the exam, you are given some instructions already and it tells you about a threat profile which you need to emulate. Meaning you have to write your own cobalt strike profile to emulate the adversary their traffic etc. The exam has 8 flags which you need to collect, with a passing grade of 6 flags. The best tip comes from RastaMouse himself, which is crucial to pass the exam. 

![RastaMouse discord pinned message](/assets/img/crto_post.png)

You get 48 hours of exam access which can be split in-between 4 days, meaning you don’t have to stay up the full 48 hours stressing that if you go to sleep you might waste valuable time you needed to pass. You can sleep peacefully and take your breaks, cook food etc, which was a nice change! You can start and stop the exam just like the lab from SnapLabs, but stopping it also means all shells will die, unless you have built in some sort of persistence. The exam environment was stable and replicates a real-world environment. There is *plenty* of time to finish the exam within the 48 hours of access. 

The exam environment is like in the labs closed off and you will need to use the tools which are on the exam. They give you a attacker kali and windows machine as like the lab, but not all tools will be there. There is a [post](https://rastamouse.me/why-tool-restricted-exams-sometimes-matter/) from RastaMouse explaining the why. You won't have BloodHound for example, and some other tools I was familiar with.

## My exam experience
10 minutes before my scheduled exam starting time my internet went down and I had to go to the office with my laptop to take the exam from there, which stressed me quite a bit. I was supposed to start at 9:30 but started around 10:30. In about 7 hours I had 6 flags which would mean I passed already, it was time to shutdown my laptop and to go home since the internet has been restored. Dammn what was I mad and stressed about my internet going down, luckily my Active Directory knowledge is strong which made me capture some flags quickly and that helped me lose the stress.

After dinner I took my time to enumerate again and got access to the 7th flag. Then I went to bed and got the 8th flag after a good night of sleep. Two more days left and 32,5 more hours of exam lab time remaining. The abuses in the exam weren't difficult except one. 
 
![Exam timeline](/assets/img/crto_exam.png)

## Conclusion
I really liked the course and the lab, although the written material was bare sometimes. I would have liked to read more about Red Teaming and the Active Directory abuses and explanations could have been expended upon. I also missed a challange part in the lab where it wouldn't take you through it and left you to figure it out on your own. 

Overall the course and the content was good and I hope RastaMouse created a advanced version.
