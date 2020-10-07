---
title:  "Discussion: Tools and Techniques for Writing Secure Code"
date:   2020-02-01 01:01:00
categories: notes lecture
layout: post
---

Topic: "Discussion: Tools and Techniques for Writing Secure Code," led
by Roger.

Required material:
  - ["Security Bugs are Fundamentally Different Than Quality Bugs"][rw_securitybugs]. Provides an overview of software testing, quality assurance, and different kinds of software bugs.
  - ["How to Cyber Security: Fuzzing does not mean random"][rw_fuzzing]. This article gives a background and introduction to fuzzing before going into modern examples. Several different fuzzing methodologies are discussed as well.
  - ["Software Security for Open Source Systems"][rw_opensource]. This paper covers several challenges to open source software and surveys enhancements that attempt to address existing security concerns for open source software.
  - [Risky Biz Podcast #406, "Making a Killing From Bug Bounty Programs"][rw_riskybugbounty]. Skip to the interview at 22:20. An interesting interview that provides and inside look at bug bounty programs, those who participate in them, and what to expect from them.
  - [Secure Code Reviews][rw_codereviewmitre].  MITRE's publication outlining the best practices to conduct code review for quality assurance.

[rw_securitybugs]:https://medium.com/bugbountywriteup/security-bugs-are-fundamentally-different-than-quality-bugs-9eb8f8663089
[rw_fuzzing]:https://securityboulevard.com/2020/05/how-to-cyber-security-fuzzing-does-not-mean-random/
[rw_opensource]:https://ieeexplore.ieee.org/abstract/document/1176994?casa_token=I6BKgyQZetcAAAAA:kqzCrw1PHYsmFd286fNu_JLTtIAC7CLJZJK2vbmRUDr-QqyejdkhcogIKjg_sKtsejevBBbM_Q
[rw_riskybugbounty]:http://media.risky.biz/RB406.mp3
[rw_codereviewmitre]:https://www.mitre.org/publications/systems-engineering-guide/enterprise-engineering/systems-engineering-for-mission-assurance/secure-code-review

Optional Material:
 - ["Github Rolls Out New Code Scanning Security Features to all Users"][rw_github]. New code scanning feature analyzes every pull request for known vulnerabilities and problematic source code and prompts users to revise code.
 - ["Bug bounties: A good tool, but don’t make them the only tool in security"][rw_bugbounty]. An article providing a high level overview of bug bounty programs and examines the pros and cons of these programs.
 - [Synopsys Code Review Glossary][rw_codereview]. Broad overview and introduction to the code reviewing process.
 - ["Why fuzzing is your friend for DevSecOps"][rw_fuzzing2]. An article demonstrating the value of effectively utilizing fuzzing when evaluating software security.

[rw_github]:https://www.zdnet.com/article/github-rolls-out-new-code-scanning-security-feature-to-all-users/?mc_cid=c55385b39c&mc_eid=dc1a5bd20c#ftag=RSSbaffb68
[rw_bugbounty]:https://www.synopsys.com/blogs/software-security/bug-bounty-programs/
[rw_codereview]:https://www.synopsys.com/glossary/what-is-code-review.html
[rw_fuzzing2]:https://gcn.com/articles/2020/05/15/fuzzing-software-vulnerability-detection.aspx


### Discussion Questions

Why are software vulnerabilities important?

Why do software bugs regularly occur?

How can we write better code?
 + What does better mean? Less bugs, more secure?
 + What tools or resources exist to address this problem?

What does fuzzing provide to enhance software security?
 + Is the complexity of the technique worth the rewards?

Is open source code secure? Can it be trusted?
 + Does more visibility into the codebase make a product more secure?

Are bug bounties helpful and worthwhile for companies to pursue?
 + Can security researchers make a living from bug bounties?
 + Are bug bounties ethical?
 + How do you ensure responsible disclosure in bug bounty programs?

Are code reviews a beneficial use of employees time?
