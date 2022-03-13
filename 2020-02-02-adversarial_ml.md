---
title:  "Discussion: Adversarial Attacks on Machine Learning Systems"
date:   2099-02-01 01:01:00
categories: notes lecture 
layout: post
excerpt_separator: <!--more-->
---

Topic: "Discussion: adversarial attacks on machine learning systems" 

<!--more-->

Required material:

 - ["Slight Street Sign Modifications Can Completely Fool Machine Learning
   Algorithms"][gg_ackerman]. A non-technical overview of a white box
adversarial attack, showing how such an attack could be carried out in the real
world. In this particular attack, researchers were able to make slight
alterations to physical street signs that interfered with algorithmic
identification, but were not obviously malicious to human observers.
 - ["This Neural Net Hallucinates Sheep"][gg_shane]. This article describes a
   particular type of errors with a single ML based computer vision system, and
explores why machine learning is prone to this kind of mistake. In particular,
it addresses one reason why ML often works well and then suddenly makes
mistakes that seem absurd to humans.
- ["Adversarial Attacks and Defences: A Survey"][gg_chakraborty]. This is a
   survey paper of recent developments in adversarial attacks on machine
learning systems and defenses that have been developed against them. It has
both technical details on how attacks and defenses are carried out (which you
should read one or two to get a feel for and then skim) as well as overviews of
which strategies are still considered effective and which ones have proven
unhelpful.

Optional materials:

 - [Researchers Fooled a Google AI Into Thinking a Rifle Was a
   Helicopter][gg_matksakis]. This article describes an attack very similar to
the street sign article, but performed on a black box system---one the
attackers had no internal access to. As you read, think about how the writer
describes the computer vision system: are they anthropomorphizing it? Is that
helpful or harmful?
 - [Watson Medical Algorithm][gg_munroe]. A visual representation of some of
   the decision making processes underlying current state of the art medical
machine learning systems. An excellent primer on the steps already being taken
to minimize harm in complex systems with potentially faulty algorithms. 
 
[gg_chakraborty]:https://arxiv.org/pdf/1810.00069.pdf
[gg_ackerman]:https://spectrum.ieee.org/cars-that-think/transportation/sensors/slight-street-sign-modifications-can-fool-machine-learning-algorithms
[gg_shane]:http://nautil.us/blog/this-neural-net-hallucinates-sheep
[gg_matksakis]:https://www.wired.com/story/researcher-fooled-a-google-ai-into-thinking-a-rifle-was-a-helicopter/
[gg_munroe]:https://xkcd.com/1619/


### Discussion Questions

What insight can we glean from the sheep article? What challenges does it reveal about working with ML?

"Security through Obscurity" is the term for security measures that rely on attackers being unfamiliar with how a defense works. For instance, splitting attacks into whitebox and blackbox attacks is mainly a StO consideration- an insider can usually perform a whitebox attack. Of the techniques presented in the survey paper, which ones fall under this umbrella?
 - In most applications, security through obscurity is discouraged since it fails badly against attackers with inside information or luck. Should that still be true for ML? In what situations?

Which attacks are caused by technical weaknesses in how ML works, and which by non-technical factors the specific application creates?
 - What could be done to address these non-technical attacks?

Consider the following use cases for machine learning:
 - A lung cancer diagnosis system
 - A credit card fraud detection system
 - A product recommendation algorithm
 - A weapons control system on a battleship
 - A shopper tracking algorithm in a no-checkout store

ML based systems are difficult to test in non-adversarial conditions. In the above cases, what safeguards would you want to ensure that ML solutions are robust? What are the consequences of safeguards failing?
 - Should there be robustness requirements? In which cases? Who checks that they are being met, and how?
 - In which cases would you, personally, demand an expert be on hand to overrule the system? Are there any where an expert should not be able to overrule the system?
  + How should safeguards be enforced? Regulation? Industry standards? Civil lawsuits? 
  + Who should be responsible, in each of these cases, for the safeguards working properly?
 - How should each case fail gracefully if it encounters something outside of its experience?
 - Which cases seem most vulnerable to attack? Which most resilient? Why?

