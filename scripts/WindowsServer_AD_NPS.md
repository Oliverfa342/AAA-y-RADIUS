# Windows Server 2022 — AD, NPS y RADIUS

## Estado documentado

Servidor `JUMP-SRV-01.redes.test` (controlador de dominio `redes.test`), con **AD DS**, **DNS** y **Network Policy and Access Services/NPS** instalados. El servicio **IAS** estaba en ejecución. El servidor fue reutilizado de una práctica anterior: no reinstalar ni quitar sus roles.

### Interfaces

- `Ethernet0` = `192.168.53.10/24`, gateway `192.168.53.2`, VMware VMnet8 NAT.
- `Ethernet1` = `10.15.71.130/29`, gateway vacío, VMware VMnet3.
- VMnet3 enlaza con Cloud1 `eth3` en GNS3 VM.

## Verificación de roles, servicios y red (PowerShell administrador)

```powershell
Get-WindowsFeature AD-Domain-Services,DNS,NPAS
Get-Service IAS
Get-ADDomain
ipconfig
netstat -ano -p udp | findstr ":1812"
netstat -ano -p udp | findstr ":1813"
```

## Creación y comprobación de grupos AD

La práctica creó grupos globales de seguridad `NIVEL_0`, `NIVEL_5` y `NIVEL_10` en `CN=Users,DC=redes,DC=test`.

```powershell
Import-Module ActiveDirectory
New-ADGroup -Name "NIVEL_0" -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=redes,DC=test"
New-ADGroup -Name "NIVEL_5" -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=redes,DC=test"
New-ADGroup -Name "NIVEL_10" -GroupScope Global -GroupCategory Security -Path "CN=Users,DC=redes,DC=test"
```

> Esos tres comandos son los usados originalmente para crear los grupos. **Si los grupos ya existen, no los vuelvas a crear**.

### Cuentas y pertenencia a grupos

La práctica utiliza `radius0`, `radius5` y `radius10`. La contraseña de AD se introdujo o restableció en Windows y cumple la complejidad del dominio; no debe publicarse.

```powershell
# Ejemplo de alta para una cuenta nueva (no ejecutar sobre cuentas existentes)
$clave = Read-Host "Contraseña nueva de radius0" -AsSecureString
New-ADUser -Name "radius0" -SamAccountName "radius0" -Path "CN=Users,DC=redes,DC=test" -AccountPassword $clave -Enabled $true -ChangePasswordAtLogon $false

# Comandos usados para añadir pertenencia
Add-ADGroupMember -Identity NIVEL_0 -Members radius0
Add-ADGroupMember -Identity NIVEL_5 -Members radius5
Add-ADGroupMember -Identity NIVEL_10 -Members radius10

# Diagnóstico real del laboratorio
Get-ADUser radius0 -Properties Enabled,LockedOut,MemberOf | Format-List Name,Enabled,LockedOut,MemberOf
Enable-ADAccount -Identity radius0
Set-ADAccountPassword -Identity radius0 -Reset -NewPassword (Read-Host "Contraseña nueva para radius0" -AsSecureString)
Set-ADUser -Identity radius0 -ChangePasswordAtLogon $false
Get-ADGroupMember NIVEL_0
Get-ADGroupMember NIVEL_5
Get-ADGroupMember NIVEL_10
```

Se detectó `Enabled: False` en `radius0`; al intentar habilitarlo Windows mostró **ADPasswordComplexityException**. Después de restablecer con una contraseña válida y habilitar la cuenta, la autenticación de `radius0` fue exitosa.

## NPS — configuración gráfica utilizada

1. Abrir `nps.msc`; registrar el servidor NPS en Active Directory.
2. **Clientes y servidores RADIUS → Clientes RADIUS**:
   - `R1-CISCO` / `10.15.71.132` (habilitado).
   - `SW1-CISCO` / `10.15.71.131` (habilitado).
   - Introducir un secreto compartido **privado** para cada cliente y hacerlo coincidir con IOS.
3. **Directivas → Directivas de red**:
   - `AAA-NIVEL-0` con condición grupo de Windows `NIVEL_0`;
   - `AAA-NIVEL-5` con `NIVEL_5`;
   - `AAA-NIVEL-10` con `NIVEL_10`.
4. Cada directiva: **Conceder acceso**, autenticación **PAP/SPAP** para esta red de laboratorio aislada (no producción), `Service-Type = NAS-Prompt`.
5. **Configuración → Atributos RADIUS → Específico del proveedor → Cisco-AV-Pair** (Cisco, vendor ID 9, atributo 1):
   - `shell:priv-lvl=0`
   - `shell:priv-lvl=5`
   - `shell:priv-lvl=10`
6. Opcional según política: ignorar propiedades de acceso telefónico de la cuenta AD para dejar el control a NPS.
7. **Cuentas**: registro de solicitudes de autenticación y contabilidad; rutas locales bajo `C:\Windows\System32\LogFiles`. Los puertos observados son UDP `1812/1645` para autenticación y `1813/1646` para cuentas.

## Firewall y registro

Permitir únicamente desde los dos Cisco los puertos de NPS; evitar reglas duplicadas:
```powershell
New-NetFirewallRule -DisplayName "LAB-RADIUS-AUTH" -Direction Inbound -Protocol UDP -LocalPort 1812 -RemoteAddress 10.15.71.131,10.15.71.132 -Action Allow
New-NetFirewallRule -DisplayName "LAB-RADIUS-ACCOUNTING" -Direction Inbound -Protocol UDP -LocalPort 1813 -RemoteAddress 10.15.71.131,10.15.71.132 -Action Allow
```

**Comandos para consultar solicitudes y el servicio:**

```powershell
Get-Service IAS
Get-WinEvent -FilterHashtable @{LogName='Security';Id=6272,6273;StartTime=(Get-Date).AddHours(-2)} -MaxEvents 20 | Format-List TimeCreated,Id,Message
Get-NpsRadiusClient
```

- **6272:** NPS concedió acceso.
- **6273:** NPS rechazó acceso.
- En el primer fallo, el evento 6273 indicó motivo **16: credenciales no coincidentes**; el cliente era R1-CISCO `10.15.71.132`.

### Respaldo NPS: privado, NO subir a GitHub

```powershell
New-Item -ItemType Directory -Path C:\RespaldoNPS -Force
Export-NpsConfiguration -Path "C:\RespaldoNPS\NPS-RADIUS.xml"
```

El XML exportado puede contener secretos RADIUS y se debe guardar fuera de repositorios públicos.

## Evidencias observadas

- Captura de `ipconfig` con Ethernet1 `10.15.71.130`.
- Captura de `redes.test\Users` con tres grupos y tres cuentas.
- Capturas NPS de dos clientes RADIUS habilitados y tres directivas.
- Atributos `shell:priv-lvl=0` y `Service-Type = NAS Prompt`.
- Eventos 6272 y 6273 en Visor de eventos.
