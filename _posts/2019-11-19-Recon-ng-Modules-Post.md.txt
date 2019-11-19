---
layout: post
title: "Cool way to OSINT your targets - My own Recon-ng implementation"
author: "Mor Davidovich @Dec0ne"
---
Hey guys,<br>This is gonna be a quick post on a cool project I've been working on.<br><br>
I take a class on Pentesting (HDE by See-Security) and my instructor gave us a project to complete. The project was to create a python program that, when given a company name, does the following:
* Collects a list of employees from LinkedIn
* Collects a list of emails posted in public websites (via google search or other search engine)
* Create a list of emails for the employees using a pattern the company is likely using (based on the public emails we found)
* Verify the which emails exists (via SMTP or any other service) - delete those that doesn't.
* Perform dictionary attack against the domain in order to find new subdomains via DNS queries.
* Go over IP C-classes the program found in the previous task and do a perform a reverse DNS resolve for all those C-Class.

<br>
We were given a free hand in regards to how we implement the project.<br> I chose to create my own modules in Recon-ng.<br>
This is the final result:<br>
<iframe width="560" height="315" src="https://www.youtube.com/embed/d-16rI4kKqc" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe><br>

It has been fun creating these modules and it gave me a pretty good idea of what is possible in regards to OSINT.<br>
The modules are available in my [GitHub](https://github.com/Dec0ne/Recon-ng-Modules/)

Until next time...<br>
Dec0ne
