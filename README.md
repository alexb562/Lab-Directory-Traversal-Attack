# Lab-Directory-Traversal-Attack
## Background
The web development team at a small ecommerce company bwapp.com has experienced several breaches with their web application in the past year, specifically involving injection attacks and directory traversals. They decided to contract Madrid Pentesting to run their first web application pen test to both remediate and secure their systems for the future, with a focus on securing credentials of users and admins. 
## Scope
The pentester was given permission to access to the web server machine running Ubuntu Linux. They used a virtual Kali box on VirtualBox to attack. The tester used the IP of the web server machine (192.168.1.48) and permission was also given to access the server using any admin credentials if found. 
## Findings
- Upon reviewing the logs that the web app team provided them, the pentester suspected that they were victims of a directory traversal attack. They noticed that the logs mentioned jumps of directories (which are injected into the url and represented by ../../../ which is the Linux command to move up directories) and it seemed as though the attacker was able to get at sensitive data. The pentester wanted to test what architecture was available from directory traversals and if there could be more vulnerabilities. 

- They first used gobuster to get a sense of directories that would be available to traverse. Using the command  gobuster dir -u http://192.168.1.48 -w /home/kali/SecLists/Discovery/Web-Content/big.txt -t 50 the pentester was able to scan the IP of the server machine, and using a prewritten list of many common directory names (found in big.txt) the search was able to return some potential endpoints to traverse.

<img width="827" height="529" alt="gobuster" src="https://github.com/user-attachments/assets/42acd0d3-e183-4658-9bfb-0eb45dd840ea" />

- They started with the /webdav directory and input that url into their browser, which returned the following:


[Alt text](webdav.png)

- Review of the documents revealed no relevant information for exploitation. However, they followed the link to the parent directory (was also be able to be accessed by using the standard directory traversal technique of adding ../) and were taken to what appeared to be the root document:

[Alt text](parent.png)

- Knowing that a portal that has something to do with admin could be useful, they followed the link to a login page to the admin portal where they were able to enter simply by entering the username of bee (no password):

[Alt text](php.png)

- However, it appears like there are no credentials here to exploit (they checked the databases with a special focus on user_privileges), so the pentester went back to look at other directories. The tables also did not reveal any sensitive information.
They then went back to the root document and went to the evil directory, which revealed many promising documents here:

[Alt text](evil.png)
- Upon looking through some of the documents, the pentester found this document labeled ssrf-3.txt. 

[Alt text](ssrf.png)

- Upon analysis the pentester realized that there are several potential entry points to continue to traverse directories further up the chain. They started with the link that traversed to bWAPP/robots.txt, which resulted in the following page:

[Alt text](portal.png)

- The pentester noted that admin credentials were listed, presenting a critical vulnerability if malicious actors were to follow the same path. They confirmed that these credentials were valid to log in. They then went back to the ssrf-3.txt document to attempt to traverse the second url listed, which ends in the directory passwords/heroes.xml, seeming to be a high value document. They found the following:

[Alt text](xml.png)

- Jumping back one directory to just /passwords, the tester found this, which leads to two other potentially valuable configuration files:

[Alt text](passwords.png)

- Upon review of the web.config.bak file, the tester was able to find what appeared to be the login for the bWAPP database as wolverine/Log@N, demonstrating weak credentials and the ability to access the wider database.

[Alt text](database.png)


## Recommendations
### PHPmyadmin brute force
The tester was able to reach the root document and access the phpmyadmin portal. Upon landing there, they only had to enter a username and click enter, to which they were entered in as a low-privilege user. Although they did not enter with any credentials, they could easily enumerate by reviewing the structure and architecture of the portal, export certain data types, and escalate privileges. This can lead to full-scale server compromise if privileges are escalated. They could also potentially lauch D/Dos attacks which could crash the server. 
### Robots.txt and ssrf-4.txt admin credentials traversal
Upon traversing through the numerous directories, the tester was able to land on the robots.txt path which exposed admin credentials. Access to these privileges can lead to a wide variety of serious consequences, including exfiltration of sensitive data, installation of malware backdoors, financial losses, and reputational damage. Further, they were able to access the aforementioned ssrf file which held usernames and passwords of users. 
### web.config.bak database exposed credentials
Navigating through the two listed configuration passwords in the /passwords directory, the tester found the login information to the wider Bwapp database. Allowing an attacker these credentials presents many of the same threats that have been listed, where both user and admin credentials can be exfiltrated. 
## Solutions
- One of the main issues of this test surrounded directory traversal, in which the tester was able to move quite freely all the way to the root document. Input validation and filtering should be implemented using tools such as OWASP Modsecurity such that attackers cannot inject any commands into the URL header. This can be done specifically with Regex filtering (common injection logic is filtered out) and parameterized queries (input is not treated as SQL input but rather just plain data). The organization should also obscure and sanitize the URL so that any injection points are not shown (this can be done with tools such as ModRewrite from Apache). Overall, this will limit the movement the tester was able to gain by freely traversing through directories and files.

- Another serious issue was the tester gaining access to the PHP admin panel through brute force without a password. This presents serious risks to users’ and employees’ data. A strong password policy should be enforced across both employees and users (the NIST framework is a good place to look for building strong credential culture and can be found at https://pages.nist.gov/800-63-4/sp800-63b/passwords/). Further, multi-factor authentication (MFA), which requires another form of authentication alongside a password, drastically lowers the risk of brute force access. 

- Another theme discussed thus far is cryptographic failure, meaning sensitive information was exposed easily. Encryption should be introduced across sensitive data (usernames, passwords, admin credentials, and any other PII) and can be done with hashing algorithms such as Argon2 and Bcrypt. Further, secure key management away from sensitive data should be implemented such that the attackers cannot gain the keys and break the encryption. Services that provide key management include AWS KMS and Azure KMS. 

- Overall, a strong focus on credential culture (including password policies and MFA), input sanitization/validation, and encryption will ensure Bwapp’s security strength will increase into the future. 
