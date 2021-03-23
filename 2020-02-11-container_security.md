---
title:  "Discussion: Container Security"
date:   2020-02-01 01:01:00
categories: notes lecture 
layout: post
---

Topic: "Discussion: Container Security" 

Required Material:

- [What is a Container?](https://www.youtube.com/watch?v=EnJ7qX9fkcU). To secure containers, it’s essential to understand what they are and their lifecycle. This is an introduction to concepts and terminology.
- [Ten layers of container security](https://www.redhat.com/en/resources/container-security-openshift-cloud-devops-whitepaper). Behind this RedHat advertisement is a comprehensive high-level take on container security. Focus on items 1 to 5, especially item 1, where it describes the mechanisms containers use for isolation.
- [A Measurement Study on Linux Container Security: Attacks and Countermeasures](https://dl.acm.org/doi/abs/10.1145/3274694.3274720). In this paper, the authors run a set of kernel exploits against containers and see which ones are blocked. Read the abstract, section 2, and section 4. The takeaway is that the primary isolation mechanisms of containers (namespaces and cgroups) are less effective at blocking these exploits than other security mechanisms (SELinux, seccomp, capabilities).
- [Malicious Docker Hub Container Images Used for Cryptocurrency Mining](https://www.trendmicro.com/vinfo/in/security/news/virtualization-and-cloud/malicious-docker-hub-container-images-cryptocurrency-mining). This article provides an example of a non-runtime attack, where the attacker uploads a malicious image to a publicly available registry, using a common name to trick developers into using it.
- [Notes on Monolithic Kernels and Virtual Machine Managers](http://web.mit.edu/6.033/2017/wwwdocs/lec/l06.pdf). These notes explain why the kernel, and therefore containers, provide a weaker security guarantee than virtual machines. The gist is that the monolithic architecture of Linux makes it easier to introduce bugs. Keep in mind that isolation is only one aspect of security.


Optional Material:

- [chroot](https://man7.org/linux/man-pages/man2/chroot.2.html), [namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html), [cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html), [capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html). Man-pages for the various kernel features that make up a container.
- [Hundreds of Vulnerable Docker Hosts Exploited by Cryptocurrency Miners](https://www.imperva.com/blog/hundreds-of-vulnerable-docker-hosts-exploited-by-cryptocurrency-miners/). An extreme example of the dangers of the Docker daemon. Since it runs as root and allows you to create containers, access to it essentially gives you root. In this case, the author finds the daemon publicly exposed on hosts throughout the Internet, and it’s no surprise that many of them are running crypto-miners.
- [Breaking out of Docker via runC – Explaining CVE-2019-5736](https://unit42.paloaltonetworks.com/breaking-docker-via-runc-explaining-cve-2019-5736/). A vulnerability in runC (the program that directly creates containers) that illustrates that root inside the container is very similar to root outside (at least for Docker). The details can be skimmed, but focus on the last part about Privileged Containers.
- [gVisor Demo](https://www.youtube.com/watch?v=TJJT8wc0T_c). gVisor is a container runtime that provides a proxy syscall interface to reduce the attack surface of the kernel for untrusted containers.
- [How AWS’s Firecracker virtual machines work](https://www.youtube.com/watch?v=BIRv2FnHJAg). Firecracker is a middle ground between containers and virtual machines, using a slimmed down VM to get fast startup times and low overhead, while keeping the security benefits of VMs.


### Discussion questions

How does a container work?
- Why is this important to its security?
- How do namespaces and cgroups provide isolation? Are there any security benefits?
- How do Linux "capabilities" affect security?
- Does it matter which user we run a container with?

Why might you choose to use a container to run programs?
- Are there any security benefits?
- Are there any deployment-related benefits?
- Why would you choose a container over a regular process or a virtual machine?

What does the attack surface look like for a container?
- How does this compare to a regular process or virtual machine?
- Do we only need to be concerned about the container during runtime, or are there other stages that could be vulnerable?
- Images are used to package container software. How could this be attacked? Is it the same for other methods of software delivery?
- A separate network layer is often used in container-based deployments. In what ways could this be vulnerable?
- Secrets like passwords and keys are often needed for containers to perform various tasks. Are there any safe ways in which they can be used?
- Why is the kernel so important for the security of containers?

How can we mitigate attacks against a container-based deployment?
- What methods can we use to harden the container?
- How do the concepts of least-privilege and layered-security apply here?
- What about other parts of the deployment pipeline, such as building the image or storing it in a registry?

Are containers suitable for a situation in which multiple users share the same hardware (i.e. multitenancy)? Why or why not?
- Can this be improved upon?

When implementing container security measures, are there any trade-offs to be made?
- In what cases would they be acceptable?

Containers are an example of a rapidly adopted and developed technology. Would you be hesitant to use this kind of technology in certain cases?
