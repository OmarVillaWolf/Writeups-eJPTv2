# Enumeración de dominio 

Tags: #Powershell #AD #Enumeration #PowerShellModule #PowerView #PowerHuntShares #SharpView 


```bash 
Flujo: 
	1. PowerShell Module → Qué existe (Ver usuarios y grupos)
	2. PowerView → Qué se puede abusar (Detectar permisos interesantes)
	3. BloodHound → Qué caminos existen (Ver un camino indirecto a Domain Admin)
	4. PowerHuntShares → Dónde hay acceso (Encontrar credenciales en un share)
	5. SharpView → Confirmar de forma sigilosa (Confirmar ACLs antes de abusar)
```


## Iniciar sesión con otro usuario en Powershell 
```powershell 
❯ whoami    # Conocer el nombre de la máquina 

# Paso 1: Crear variable de contraseña segura
❯ $passwd = ConvertTo-SecureString 'P@ssWord123!' -AsPlainText -Force

# Paso 2: Crear objeto de credenciales
❯ $creds = New-Object System.Management.Automation.PSCredential ('PC_name\admin', $passwd)

# Paso 3: Conectar a máquina remota (Enter-PSSession)
❯ Enter-PSSession -ComputerName PC_name -Credential $creds
```

## Powershell Module (ActiveDirectory Module)

* [PowerShell AD Module](https://learn.microsoft.com/en-us/powershell/module/activedirectory/?view=windowsserver2025-ps)
* [ADModule](https://github.com/samratashok/ADModule)

```bash 
El módulo nativo de Active Directory de Microsoft, utilizado por administradores para gestionar y consultar el dominio mediante PowerShell, ya que es una base limpia y legítima de la enumeración interna AD.

Qué permite enumerar:
	- Usuarios del dominio
	- Grupos y membresías
	- Equipos del dominio
	- Atributos de objetos AD
	- Relaciones básicas entre usuarios, grupos y equipos
	- Información de Kerberos (SPNs, cuentas de servicio)

Es importante porque permite realizar enumeración legítima usando herramientas propias del sistema, lo que:
	- Reduce ruido frente a herramientas ofensivas
	- Simula acciones reales de un administrador
	- Funciona incluso cuando herramientas ofensivas están bloqueadas

Ejemplo típico:
	- Enumeras todos los usuarios con Get-ADUser
	- Identificas un usuario interesante (servicio, admin, legacy)
	- Revisas grupos y membresías
	- Detectas posibles vectores de escalada o movimiento lateral
```

```powershell 
❯ Import-Module C:\AD\Microsoft.ActiveDirectory.Management.dll    # Importar el módulo
❯ Import-Module C:\AD\ActiveDirectory.psd1    # Importar el módulo
```

```powershell 
❯ Get-ADDomain                             # Obtener las características del dominio actual 
❯ Get-ADDomain -Identity domain1.corp      # Obtener las características de otro dominio 

❯ (Get-ADDomain).DomainSID                 # Obtener el SID del dominio actual 

❯ (Get-DomainPolicyData).systemaccess      # Politicas del dominio actual 
❯ (Get-DomainPolicyData -domain domain1.corp).systemaccess     # Mirar las politicas de otro dominio 

❯ Get-ADDomainController                   # Obtener los controladores del dominio actual
❯ Get-ADDomainController -DomainName domain1.corp -Discover    # Obtener los controladores de otro dominio 

❯ Get-ADUser -Filter * -Properties *       # Obtener lista de usuarios del dominio actual 
❯ Get-ADUser -Identity user1 -Properties *
# Obtener todas las propiedades de los usuarios del dominio actual 
❯ Get-ADUser -Filter * -Properties * | select -First 1 | Get-Member -MemberType *Property | select Name  
❯ Get-ADUser -Filter * -Properties * | select name,logonaccount,@{expression={[datatime]::fromFileTime($_.pwdlastset)}}
# Buscar una palabra en particular en los atributos del usuario 
❯ Get-ADUser -Filter 'Description -like "*built*"' -Properties Description | select name,Description 

❯ Get-ADComputer -Filter * | select Name    # Obtener una lista de computadores del dominio actual
❯ Get-ADComputer -Filter * -Properties * 
❯ Get-ADComputer -Filter 'OpertingSystem -like "*Server 2022*"' -Properties OperatingSystem | select Name,OperatyngSystem
❯ Get-ADComputer -Filter * -Properties DNSHostName | %{Test-Connection -Count 1 -ComputerName $_.DNSHostName}

❯ Get-ADGroup -Filter * | select Name     # Obtener todos los grupos del dominio actual
❯ Get-ADGroup -Filter * -Properties *
❯ Get-ADGroup -Filter 'Name -like "*admin*"' | select Name   # Obtener todos los grupos conteniendo la palabra 'admin'

❯ Get-ADGroupMember -Identity "Domain Admins" -Recursive    # Obtener todos los miembros del grupo "Domain Admins"
❯ Get-ADPrincipalGroupMembership -Identity user1            # Obtener el 'group membership' de un usuario


```

## PowerView  

* [PowerView](https://github.com/ZeroDayLab/PowerSploit/blob/master/Recon/PowerView.ps1)

```bash 
PowerView es un módulo de PowerShell para enumeración completa de Active Directory.

Qué te permite ver:
	- Usuarios, grupos y equipos
	- Membresías de grupos
	- ACLs (quién puede hacer qué sobre qué)
	- Delegaciones
	- Sessions, logged-on users
	- Kerberos (SPNs, AS-REP)
	- Relaciones de confianza

Por qué es clave:
PowerView es lo que te permite entender el dominio como un grafo, no como una lista de máquinas. Es como un mapa del dominio. 

Ejemplos de preguntas que responde:
	- ¿Qué usuario tiene permisos peligrosos?
	- ¿Quién puede resetear contraseñas?
	- ¿Hay ACLs mal configuradas?
	- ¿Desde dónde puedo moverme?
```

```powershell 
❯ . C:\AD\PowerView.ps1               # Cargar PowerView en memoria 
❯ Import-Module .\PowerView.ps1       # Importar el módulo 
```

### Enumeración AD 

```powershell 
❯ Get-Domain                          # Obtener las características del dominio actual 
❯ Get-Domain -Domain domain1.corp     # Obtener las características de un dominio en específico


❯ Get-DomainSID                       # Obtener el security identifier del dominio actual 'SID'


❯ Get-DomainPolicyData                # Politicas del dominio actual 
	- LockoutBadCount : 0        # No hay política de lockout y se puede hacer Password spraying, AS-REP roasting + cracking sin miedo
	- MinimumPasswordLength : 7  # Corto para hacer Kerberoasting, AS-REP roasting, Spraying
	- MaxTicketAge : 10          # Relevante para Pass-the-Ticket, Golden Ticket


❯ Get-DomainController            # Info del DC como la IP, OS Version, name, etc... 
❯ Get-DomainController -Domain domain1.corp    # Info de un DC en específico  


❯ Get-DomainUser                               # Listar toda la info de los usuarios del DC
❯ Get-DomainUser -Identity student1        
❯ Get-DomainUser | select samaccountname       # Listar todos los usuarios en el DC sin sus propiedades
# Obtener todas las propiedades de los usuarios del dominio actual 
❯ Get-DomainUser -Identity user1 -Properties *
❯ Get-DomainUser -Properties samaccountname,logonCount    # Detectar cuentas activas 
	- logonCount > 0     # La cuenta si se ha usado 
	- logonCount = 0     # Posible: Cuenta nueva, cuenta vieja nunca usada, service account
# Buscar una palabra en particular en los atributos del usuario 
❯ Get-DomainUser -LDAPFilter "Description=*built*" | select name,Description   
❯ Find-DomainUserLocation    


❯ Get-DomainComputer | select Name     # Obtener una lista de computadores del dominio actual  
❯ Get-DomainComputer | select dnshostname, logonCount   # Lista de computadoras (objetos) conectados al DC, ademas muestra el loggeo para detectar cuentas activas 
❯ Get-DomainComputer | select -ExpandProperty dnshostname   # Obtener los equipos del dominio
❯ Get-DomainComputer -OperatingSystem "*Server 2022*"
❯ Get-DomainComputer -Ping 


❯ Get-DomainGroup                      # Obtener toda la info de los grupos 
❯ Get-DomainGroup | select Name        # Obtener todos los grupos del dominio actual 
❯ Get-DomainGroup -Domain <targetdomain>
❯ Get-DomainGroup *admin*              # Obtener todos los grupos que contienen la palabra 'admin' 
❯ Get-DomainGroup *admin* | select Name 
❯ Get-DomainGroup *admin* -Domain domain1.corp | select Name    # Obtener los grupos que pertenecen a un dominio en específico  


❯ Get-DomainGroupMember -Identity "Domain Admins" -Recurse      # Obtener la info y los miembros del grupo "Domain Admins"
❯ Get-DomainGroup -UserName "student1"     # Obtener el 'group membership' de un usuario

❯ Find-DomainShare -CheckShareAccess       # Buscar shares accesibles en el dominio 
```

### Enumeración del dominio Local 

```bash 
❯ Get-NetLocalGroup -ComputerName dcorp-dc   # Lista todos los grupos locales del dominio en una máquina (Necesita privilegios de admin en 'non-dc machines')
❯ Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators   # Obtener los miembros del grupo local 'Administrators' del dominio en una máquina (necesita privilegios de admin en 'non-dc machines')


❯ Get-NetLoggedon -ComputerName dcorp-adminsrv          # Mirar logeo activos de usuarios en una computadora (necesita privilegios locales de admin en el target)
❯ Get-NetLoggedonLocal -ComputerName dcorp-adminsrv     # Obtener el logeo de usuarios localmente en una computadora (se necesita registro remoto en el target - started by-default on server OS)
❯ Get-NetLoggedon -ComputerName dcorp-adminsrv          # Obtener el ultimo logeo del usuario en una computadora (se necesita derechos administrativos y registro remnoto en el target)


❯ Invoke-ShareFinder -Verbose    # Encontrar recursos compartidos en un host del dominio actual
❯ Invoke-FileFinder -Verbose     # Encontrar archivos sensibles en una computadora del dominio
❯ Get-NetFileServer              # Obtener todos los servidores de archivos del dominio
``` 

## PowerHuntShares 

* [PowerHuntShares](https://github.com/NetSPI/PowerHuntShares)

```bash 
PowerHuntShares es una herramienta enfocada solo en shares SMB dentro del dominio. Es muy ruidosa. Sirve para descubrir: 'recursos compartidos, archivos sensibles, ACLs para recursos compartidos, redes, computadoras, identidades, etc...' y te genera un reporte HTML.

Qué busca:
	- Shares accesibles
	- Shares con permisos de escritura
	- Shares con archivos interesantes:
		- Scripts
		- Credenciales
		- Backups
		- Configs

Es importante porque muchísimos dominios caen por:
	- Scripts con passwords
	- Archivos olvidados
	- Shares mal protegidos

Ejemplo típico:
	- Encuentras un .ps1
	- Dentro hay credenciales
	- Escalas o te mueves lateralmente
```

```powershell 
❯ Import-Module PowerHuntShares.psm1        # Importar el módulo  
```

```powershell 
❯ Invoke-HuntSMBShares -NoPing -OutputDirectory C:\Users\user\ -HostList C:\Users\user\servers.txt 

Donde:
	- Si no se remueven los DCs en el archivo 'servers.txt' este será muy ruidoso ya que vas a generar tráfico SMB innecesario hacia el servidor más monitoreado del dominio. 
	- El reporte también incluye 'ShareGraph', que puede usarse para explorar las relaciones de recursos compartidos en tu máquina host (no en la VM del estudiante).
```

```powershell 
# Escaneo completo (enumera TODO el dominio) <-- Mejor opción 
❯ Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\user\Desktop
```

```powershell 
# Escanear tu propio equipo 
❯ $env:COMPUTERNAME | Out-File C:\Users\user\myserver.txt
❯ Invoke-HuntSMBShares -NoPing -OutputDirectory C:\Users\user\ -HostList C:\Users\user\myserver.txt 
```

## PowerView 
```powershell 
❯ Get-DomainComputer | select -ExpandProperty dnshostname   # Obtener los equipos del dominio, despues guardarlos en un archivo llamada 'servers.txt'

❯ Get-DomainComputer |
Where-Object {$_.dnshostname -notlike "*-dc*"} |
Select-Object -ExpandProperty dnshostname > C:\Users\user\servers.txt      
# Filtrado 

❯ Get-DomainComputer |
Where-Object {$_.useraccountcontrol -notmatch "SERVER_TRUST_ACCOUNT"} |
Select-Object -ExpandProperty dnshostname > C:\Users\user\servers.txt      
# Mejor filtro 
```

## SharpView

* [SharpView](https://github.com/tevora-threat/SharpView)

```bash 
SharpView es la versión en C# de PowerView y es más pensado en OPSEC.

Diferencias importantes:
	- No usa PowerShell scripting directamente
	- Menos ruido en entornos con AMSI / logging
	- Más OPSEC-friendly

Cuándo usarlo:
	- Cuando PowerShell está restringido
	- Cuando quieres evitar:
		- Script Block Logging
		- AMSI

Enumera prácticamente lo mismo que PowerView:
	- Usuarios
	- Grupos
	- ACLs
	- Delegaciones
	- Sessions
```