TryHackMe — Pickle Rick Writeup

An authorized lab walkthrough covering web enumeration, command execution, privilege context, and ingredient discovery.


Item	Details: Platform	TryHackMe

Room	Pickle Rick : Difficulty	Easy

Target:	<MACHINE_IP>

Room URL: 	https://tryhackme.com/room/picklerick

1.Port Scanning

Start with a basic Nmap scan to identify reachable TCP services on the target:

bash
nmap <MACHINE_IP>

<img width="1097" height="331" alt="image" src="https://github.com/user-attachments/assets/77c894c8-b238-4c46-81e1-d3fdd49610dd" />
The scan identified two open ports: 22/tcp (SSH) and 80/tcp (HTTP). Since the challenge presents a web application, I prioritized the HTTP service for further investigation.

2. Web

I opened the web application and inspected the HTML source. A developer comment exposed a username that could be used for authentication.

<img width="1905" height="956" alt="image" src="https://github.com/user-attachments/assets/996f33a8-9272-4251-8e9c-a22123fb2d32" />

Username: R1ckRul3s 

Directory Brute-Forcing

I used Gobuster with PHP, HTML, and TXT extensions to identify resources not linked directly from the home page:


gobuster dir -u http://<MACHINE_IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt


<img width="793" height="562" alt="image" src="https://github.com/user-attachments/assets/a1cc8d10-777a-4ce5-9fc8-153a3523cba2" />

The results included login.php, robots.txt, and portal.php. The 302 redirect from portal.php to login.php indicated that the command portal required authentication.

robots.txt

I inspected robots.txt before attempting to log in. Instead of normal crawler directives, the file contained a standalone string, which I treated as a password candidate for the username discovered in the page source.

<img width="871" height="283" alt="image" src="https://github.com/user-attachments/assets/a5886990-18d0-48c6-9e54-c9bffcced07b" />

Password candidate: Wubbalubbadubdub

3. Authentication

I submitted the discovered username and password on login.php. Authentication succeeded and the application redirected me to the command panel at portal.php.

Credentials: R1ckRul3s / Wubbalubbadubdub

<img width="1911" height="838" alt="image" src="https://github.com/user-attachments/assets/3cc22bf3-fc21-46e4-8f1b-94be988315f7" />

4. Command Execution & First Ingredient

The authenticated portal accepted operating-system commands. I first listed the contents of the current web directory:

ls

<img width="1803" height="599" alt="image" src="https://github.com/user-attachments/assets/a32abf46-4690-4744-a04c-a484c4bcceee" />

The listing revealed Sup3rS3cretPickl3Ingred.txt. I used less to display its contents:

<img width="1510" height="395" alt="image" src="https://github.com/user-attachments/assets/58f304fd-cc78-4b4d-9b16-28fb61d7a59c" />

First ingredient: mr. meeseek hair

5. Locating the Second Ingredient

After finding the first ingredient in the web directory, I enumerated the root filesystem and then followed the user directories under /home:

ls /

<img width="1500" height="657" alt="image" src="https://github.com/user-attachments/assets/4dffc37d-2268-44ee-888d-55202ddf1a23" />

ls /home 

<img width="1538" height="471" alt="image" src="https://github.com/user-attachments/assets/03421824-ae6e-4600-948c-474e5733fa97" />

ls /home/rick 

<img width="1539" height="319" alt="image" src="https://github.com/user-attachments/assets/1623a7bc-bedd-4d4b-a832-831c55272c84" />

The filename contains a space, so I enclosed it in double quotes to ensure the shell treated the complete name as one argument: 

less /home/rick/"second ingredients" 

<img width="1683" height="368" alt="image" src="https://github.com/user-attachments/assets/abfa5e89-0756-4fac-90db-cabd22af4de3" />

Second ingredient: 1 jerry tear

6. Privilege Context & Final Ingredient

Before accessing /root, I checked which operating-system account was executing commands through the web portal:

whoami

<img width="1501" height="337" alt="image" src="https://github.com/user-attachments/assets/cca825ce-2f7c-4d73-9d57-5c5af8fb8431" />

The output was www-data, confirming that commands were running as the restricted web-service account.

To determine whether this account could execute commands with elevated privileges, I tested access to root's home directory:

sudo ls /root

<img width="1538" height="333" alt="image" src="https://github.com/user-attachments/assets/ffe3083a-89ef-4c50-9eec-e87f5f0ccbcf" />

The command executed successfully without requesting a password, demonstrating that www-data had passwordless sudo access. (Note: a full sudo -l output was not captured, so the exact scope of the account's sudo permissions was not verified.)

The directory contained 3rd.txt, which I read with elevated privileges:
sudo less /root/3rd.txt

<img width="1530" height="334" alt="image" src="https://github.com/user-attachments/assets/b208b6de-b395-413e-9b3b-c45bf04ee254" />

Final ingredient: fleeb juice

7. Answer Summary

The three room questions were answered in the order the ingredients were discovered:

#	Evidence Location	Answer
1st ingredient 	Sup3rS3cretPickl3Ingred.txt  (web directory) 	mr. meeseek hair

2nd ingredient	 /home/rick/second   	1 jerry tear

3rd (final) ingredient 	/root/3rd.txt	 fleeb juice
