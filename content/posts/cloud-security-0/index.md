+++
title = 'Hands on Cloud Pentesting: Zero to Hero with 0$'
date = 2025-05-16T15:31:23+05:30
draft = true
series = "Hands on Cloud Pentesting"
tags = ["cloud","pentesting","aws"]
+++

<!--more-->

## Introduction

I have been wanting to learn cloud Pentesting for a while now and struggled to find resources/courses which are affordable and provides hands on practise. Coming from a CTF background I really enjoy challenge based learning and I have found a few resources online which meet the criteria of no(low) cost to entry and hands on keyboard experience. Through this blog which may become a series I hope to keep track of my progress and also share what I learn for others who are on the same boat.

AWS is what I will be starting with since it has the highest market share right now and most of the resources/content cover AWS. I do plan on checking out Azure and GCP in the future. As far as my current knowledge with AWS pentesting goes it is near zero. I have interacted with a few S3 challenges that have been in CTF challenges in the past but apart from that the cloud knowledge I have revolves around deploying and handling servers(VM's, EC2's, Compute Engines whatever they're called). The first course I did to cover the base knowledge is [AWS Cloud Practitioner Essentials](https://explore.skillbuilder.aws/learn/courses/134/aws-cloud-practitioner-essentials/lessons) which is a free course provided by AWS covering a lot of the basic "what is cloud" and a few services provided by AWS(EBS, S3, EFS, RDS etc.). The course is purely theoretical but helps bulid a base.

Coming to hands-on AWS pentesting, popular training platforms like [tryhackme](https://tryhackme.com/cloud-access) and [hackthebox](https://www.hackthebox.com/business/professional-labs/cloud-labs-blacksky) have their selections of cloud-focused labs but they're behind a pretty steep paywall and meant for "business plans". Tryhackme did collaborate with a creator [Tyler Ramsbey](https://www.youtube.com/@TylerRamsbey) who has released the streams of him going over the AWS pentesting course(pathway) for free on [youtube](https://youtu.be/JUO-m5ga-gc?si=pm1ilRVikeOdcPiO) but since the labs are still not accessible to the public, it doesn't qualify as "hand-on" learning for me, albeit the streams are pretty good I'd recomment it to anyone wanting a more structured path(which is available for free). Tyler Ramsbey's channel also led me to find [Cloudgoat](https://github.com/RhinoSecurityLabs/cloudgoat), [Pwnedlabs](https://pwnedlabs.io/) and [Flaws 2](http://flaws2.cloud/) which are free* platforms providing labs and scenarios that you can set up locally or with a free account to learn cloud pentesting.

---

## Cloudgoat
![cloudgoat](image.png)

I want to start off with [CloudGoat](https://github.com/RhinoSecurityLabs/cloudgoat) which, according to their website is, "Rhino Security Labs' "Vulnerable by Design" cloud deployment tool. It allows you to hone your cloud cybersecurity skills by creating and completing several "capture-the-flag" style scenarios. Each scenario is composed of cloud resources arranged together to create a structured learning experience. Some scenarios are easy, some are hard, and many offer multiple paths to victory. As the attacker, it is your mission to explore the environment, identify vulnerabilities, and exploit your way to the scenario's goal(s)." This was intriguing to me as it provided sort of a "locally" deployed learning platform to learn cloud pentesting, kind of like DVWA for Web apps. The term "locally" is not exactly correct but everything is deployed on your own aws account and you decide which lab/scenario to spin up and you have complete control over it. But yes, it does require an AWS account(Azure too if you want to try that but I'll not be doing that now) to get AWS access keys which are needed to deploy each scenario. So without any further ado, lets get started.


### Setup and Installation

Being completely new to AWS and to avoid any unwanted charges to my card when setting up AWS I followed a step by step guide I found at [https://github.com/GainSec/CloudGoatTutorial](https://github.com/GainSec/CloudGoatTutorial). The setup process walks you through creating Users, User Groups, modifying policies etc. using the GUI.

Once AWS is setup and you have access keys, the next step is to install cloudgoat and add the access keys. To install cloudgoat just follow the instructions at [https://github.com/RhinoSecurityLabs/cloudgoat?tab=readme-ov-file#quick-start](https://github.com/RhinoSecurityLabs/cloudgoat?tab=readme-ov-file#quick-start). Fow now I am only going to configure aws with `cloudgoat config aws` and will leave azure for later. And that's it; cloudgoat is setup and ready to go.