# bWAPP Directory Exposure & Sensitive Credential Disclosure Lab

## Background

This project was conducted in an authorized lab environment using bWAPP, an intentionally vulnerable web application designed for security testing and education.

For the purposes of the scenario, the web development team of a small e-commerce company had experienced several security incidents involving its web application, including suspected injection and directory traversal attacks. The company contracted a penetration testing team to assess the application, identify additional vulnerabilities, and recommend remediation measures, with particular attention given to protecting user and administrative credentials.

## Scope

The assessment was performed against an Ubuntu Linux web server hosting the bWAPP application at `192.168.1.48`. Testing was conducted from a Kali Linux virtual machine running in VirtualBox.

The tester was authorized to enumerate the web application, investigate accessible directories and files, test for directory traversal and related vulnerabilities, and validate any administrative credentials discovered during the assessment.

Testing was limited to the designated lab server and was performed within an isolated and authorized environment.

## Tools and Technologies

- Kali Linux
- Ubuntu Linux
- bWAPP
- Gobuster
- SecLists
- phpMyAdmin
- VirtualBox

## Findings

### Initial Log Review

* Upon reviewing the logs provided by the web application team, the pentester identified activity consistent with a possible directory traversal attack. Several requests contained repeated parent-directory references such as `../../../`, which can be used to reference locations higher in a directory structure when an application improperly handles user-controlled file paths.

* The logs suggested that an attacker may have attempted to access files outside the application's intended directory. Based on these findings, the pentester decided to investigate the application's exposed directory structure and determine whether additional files or sensitive information could be accessed.

### Web Directory Enumeration

* The pentester first used Gobuster to enumerate directories and files exposed by the web server. The goal was to identify potentially interesting web paths that could be investigated further for misconfigurations, sensitive files, or other vulnerabilities.

* The following command was used:

```bash
gobuster dir -u http://192.168.1.48 -w /home/kali/SecLists/Discovery/Web-Content/big.txt -t 50
```

* This command instructed Gobuster to scan the target web server using the `big.txt` wordlist from SecLists while running 50 concurrent threads. The scan returned several accessible endpoints that warranted further investigation.

<img width="827" height="529" alt="Gobuster directory enumeration results" src="https://github.com/user-attachments/assets/42acd0d3-e183-4658-9bfb-0eb45dd840ea" />

### Exposed WebDAV Directory Listing

* One of the endpoints identified during enumeration was `/webdav`. The pentester navigated to this path in the browser and found that directory listing was enabled, allowing the contents of the directory to be viewed directly.

<img width="1005" height="387" alt="WebDAV directory listing" src="https://github.com/user-attachments/assets/12fd76b7-179e-4621-9930-1660c22c63a0" />

* The exposed files were reviewed for potentially sensitive information. No immediately useful credentials or configuration data were identified within the documents shown in this directory.

* The directory listing also exposed a parent-directory link. Following this link allowed the tester to navigate upward within the web-accessible directory structure and continue enumerating additional content.

<img width="563" height="403" alt="Parent directory listing" src="https://github.com/user-attachments/assets/1bdd1488-acc3-4328-9adb-550d2df5a6fb" />

### Insecure phpMyAdmin Authentication

* While reviewing the exposed directory structure, the pentester identified a link leading to a phpMyAdmin login portal. Because administrative interfaces are high-value targets, the tester attempted to determine whether the portal was properly protected.

* The tester entered the username `bee` without providing a password and was granted access to the phpMyAdmin interface. This demonstrated that the account could authenticate without a password, indicating an insecure authentication configuration.

<img width="1128" height="439" alt="phpMyAdmin access using the bee account without a password" src="https://github.com/user-attachments/assets/f5a34102-dea5-4812-bfa6-053497a9ae6b" />

* Once authenticated, the tester reviewed the accessible databases and tables, with particular attention given to `user_privileges`, to determine whether additional credentials or sensitive information were exposed.

* No immediately useful credentials were identified within the accessible database content. However, unauthorized access to a database administration interface exposed information about the application's database structure that could assist an attacker in further reconnaissance or exploitation.

### Exposed Internal Paths

* The tester then returned to the exposed web directory structure and investigated the `/evil` directory, which contained several potentially interesting files.

<img width="662" height="613" alt="Contents of the exposed evil directory" src="https://github.com/user-attachments/assets/d945198a-ab04-4ec3-ad48-7dfc95e81b5c" />

* During this review, the pentester identified a file named `ssrf-3.txt`.

<img width="1092" height="283" alt="Contents of ssrf-3.txt" src="https://github.com/user-attachments/assets/ef849d31-a6f8-406e-bc4e-eaa33e9a94bb" />

* The contents of `ssrf-3.txt` exposed several internal file paths that warranted further investigation.

* Although the filename referenced SSRF, the evidence observed during this portion of the assessment primarily demonstrated sensitive path disclosure rather than confirming a server-side request forgery vulnerability.

### Administrative Credentials Exposed in `robots.txt`

* The tester followed the first exposed path, which referenced `bWAPP/robots.txt`. Accessing the file revealed sensitive information that included administrative credentials.

<img width="1352" height="581" alt="Sensitive information exposed through robots.txt" src="https://github.com/user-attachments/assets/ab430727-42cc-422c-8d4f-984d9a3b7628" />

* The pentester confirmed that the exposed administrative credentials were valid by successfully authenticating with them.

* This demonstrated that valid administrative credentials were stored in a publicly accessible web file, creating a significant risk of unauthorized administrative access if discovered by a malicious actor.

### Sensitive Credential File Exposure

* The tester then returned to `ssrf-3.txt` and investigated another exposed path referencing `passwords/heroes.xml`.

* The file was accessible and contained additional sensitive credential information.

<img width="475" height="594" alt="Exposed heroes.xml credential file" src="https://github.com/user-attachments/assets/554f5b59-d694-4740-9da9-b4138108d513" />

* After identifying the exposed `heroes.xml` file, the tester navigated to the parent `/passwords` directory.

* Directory listing was enabled, revealing additional files that appeared to contain configuration or credential-related information.

<img width="1003" height="299" alt="Contents of the exposed passwords directory" src="https://github.com/user-attachments/assets/939f51da-f61c-4eb7-9f9c-748dcd0e8bb3" />

### Database Credentials Exposed in Backup Configuration File

* One of the exposed files, `web.config.bak`, contained database authentication information associated with the bWAPP application.

* The backup configuration file stored a database username and password in plaintext, allowing anyone with access to the file to obtain the credentials.

<img width="1003" height="400" alt="Database credentials exposed in web.config.bak" src="https://github.com/user-attachments/assets/d6006cc0-835a-4719-950d-0717e71e0e07" />

* The exposure of a backup configuration file containing plaintext database credentials represents a serious security weakness.

* An attacker who obtained valid database credentials could attempt to authenticate directly to the database. The resulting impact would depend on the permissions assigned to the exposed database account and whether the database service was reachable from the attacker's location.

### Findings Summary

Taken together, the assessment identified several weaknesses that significantly increased the application's attack surface:

* Publicly accessible directory listings.
* Exposure of internal application paths.
* Passwordless access to a phpMyAdmin account.
* Administrative credentials stored in a publicly accessible file.
* Sensitive credential files exposed through the web server.
* A backup configuration file containing plaintext database credentials.

These weaknesses could provide an attacker with information and credentials that could be used to obtain additional unauthorized access to the application or its supporting infrastructure.


## Recommendations

### 1. Disable Unnecessary Directory Listing

Directory listing should be disabled on web-accessible directories unless it is explicitly required for application functionality. Exposed directory indexes allowed the tester to browse files and identify additional sensitive resources that would otherwise have been more difficult to discover.

The web server should be configured to prevent automatic directory indexing, and sensitive directories should not be accessible directly through the web root.

### 2. Restrict Access to Administrative Interfaces

The phpMyAdmin interface should not permit authentication without a password. All administrative accounts should require strong authentication, and unused or unnecessary accounts should be disabled.

Access to phpMyAdmin and similar administrative interfaces should also be restricted where possible. This could include limiting access to trusted management networks, requiring VPN access, or applying IP-based restrictions.

Multi-factor authentication should be enabled for administrative access when supported.

### 3. Remove Credentials From Publicly Accessible Files

Administrative credentials should never be stored in files that are accessible through the web server, including files such as `robots.txt`.

The exposed credentials should be considered compromised and immediately rotated. The organization should also review other publicly accessible files to determine whether additional passwords, API keys, database credentials, or other secrets are present.

Files intended to provide instructions to search engines should not be treated as a security mechanism. Information listed in `robots.txt` can be viewed by anyone who requests the file.

### 4. Protect Sensitive Application Files

Sensitive files such as `heroes.xml`, configuration files, backup files, and credential-related documents should not be stored in publicly accessible directories.

Where these files are required by the application, access should be restricted using appropriate filesystem permissions and web server configuration. Files containing sensitive information should be stored outside of the application's public web root whenever possible.

### 5. Remove Backup Configuration Files From the Web Root

Backup files such as `web.config.bak` should not be deployed to publicly accessible directories.

Backup configuration files can contain the same secrets and settings as production configuration files and may be served as plain text instead of being processed by the application.

The exposed database credentials should be rotated, and the database account should follow the principle of least privilege so that it has access only to the resources required by the application.

### 6. Implement Secure Secrets Management

Database passwords, API keys, and other application secrets should not be hard-coded into publicly accessible configuration files.

Secrets should instead be stored using an appropriate secrets-management mechanism or protected configuration system. Access to those secrets should be limited to the application and administrators who require them.

Credential rotation procedures should also be established so that exposed or outdated secrets can be replaced quickly.

### 7. Strengthen Password and Authentication Policies

Administrative and user accounts should follow a strong credential policy. Passwordless administrative access should not be permitted.

Passwords should be stored using a dedicated password-hashing algorithm such as Argon2id or bcrypt rather than plaintext or reversible encryption.

Multi-factor authentication should be implemented for administrative accounts where possible to reduce the impact of stolen or guessed passwords.

### 8. Prevent Path Traversal

Applications should avoid using user-controlled input directly when constructing filesystem paths.

Where file access based on user input is required, the application should use an allowlist of permitted files or map user-supplied identifiers to predetermined server-side resources.

Paths should be normalized and validated before use, and the application should verify that the requested resource remains within the intended directory.

Filesystem permissions should also follow the principle of least privilege so that the web application cannot access files that are not required for normal operation.

A web application firewall such as ModSecurity may provide additional protection against common traversal attempts, but it should be treated as a defense-in-depth control rather than a replacement for secure application logic.

## Overall Remediation Priorities

The highest-priority remediation actions should include:

1. Rotating all credentials exposed during the assessment.
2. Removing sensitive and backup files from publicly accessible directories.
3. Correcting the passwordless phpMyAdmin authentication configuration.
4. Disabling unnecessary directory listing.
5. Restricting access to administrative interfaces.
6. Implementing secure secrets management and least-privilege access controls.
7. Reviewing application file-handling logic for path traversal vulnerabilities.

Together, these changes would significantly reduce the likelihood that exposed files, weak authentication controls, or improperly handled file paths could be used to gain unauthorized access to the application or its supporting infrastructure.

## Skills Demonstrated

- Web content and directory enumeration
- Analysis of exposed web resources
- Identification of insecure authentication
- Sensitive credential discovery
- Web server misconfiguration analysis
- Security impact assessment
- Remediation planning
- Technical penetration-testing documentation