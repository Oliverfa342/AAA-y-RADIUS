# Pruebas y comandos de la práctica AAA + RADIUS

**En las consolas Cisco, usar cuenta local administradora de nivel 15** cuando se necesite mostrar el running-config. Las cuentas `radius0/5/10` tienen permisos restringidos.

## R1-AAA — ejecución

```cisco
ping 10.15.71.130
ping 10.15.71.131
show ip interface brief
show aaa servers
show radius statistics
show ip ssh
show running-config
show logging
show line
show privilege
test aaa group radius radius0 <CONTRASENA_PRIVADA_AD> new-code
test aaa group radius radius5 <CONTRASENA_PRIVADA_AD> new-code
test aaa group radius radius10 <CONTRASENA_PRIVADA_AD> new-code
copy running-config startup-config
show startup-config
```

## SW1-AAA — ejecución

```cisco
ping 10.15.71.130
ping 10.15.71.132
show ip interface brief
show vlan brief
show interfaces status
show aaa servers
show radius statistics
show ip ssh
show running-config
show logging
show line
show privilege
test aaa group radius radius0 <CONTRASENA_PRIVADA_AD> new-code
test aaa group radius radius5 <CONTRASENA_PRIVADA_AD> new-code
test aaa group radius radius10 <CONTRASENA_PRIVADA_AD> new-code
copy running-config startup-config
show startup-config
```

**Resultados observados:** R1 devolvió `User successfully authenticated` con Cisco-AV-Pair 0, 5 y 10. SW1 devolvió `User successfully authenticated` con Cisco-AV-Pair 10; también se obtuvo autenticación de radius0 con privilegio 0 en otra sesión. Son pruebas RADIUS; no equivalen automáticamente a todos los privilegios de sesión.

## SSH de administradores desde Windows Server / PowerShell

La imagen IOS clásica usa algoritmos SSH heredados. Para el laboratorio se usaron opciones por **conexión**, sin debilitar globalmente Windows:

```powershell
# R1
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oMACs=+hmac-sha1 -oCiphers=+aes128-cbc adminlocal@10.15.71.132

# SW1
ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 -oHostKeyAlgorithms=+ssh-rsa -oMACs=+hmac-sha1 oliver@10.15.71.131

# Solo tras reconstruir uno de estos equipos y verificar la nueva huella SSH:
ssh-keygen -R 10.15.71.132
ssh-keygen -R 10.15.71.131
```

**R1:** usuario `adminlocal`, nivel 15; **SW1:** `oliver`, nivel 15. Las contraseñas no se incluyen.

## Prueba de privilegio real por consola

1. Abrir primero el SSH administrativo de respaldo y **dejarlo abierto**.
2. Comprobar que `line con 0` está asociada a las listas AAA; en R1 también `line aux 0`.
3. Terminar la sesión previa de consola e iniciar otra con `radius10`.
4. Ejecutar `show privilege`; resultado buscado: 10.
5. Repetir para `radius5`; en nivel 0 algunos comandos pueden no estar permitidos.
6. **NO afirmar como confirmado el nivel de sesión si no se dispone de la captura de ese inicio de sesión.**
7. Usar administradores locales para `show running-config`; un usuario de nivel 10 no puede ejecutar normalmente `configure terminal` ni `show running-config`.

## Windows Server

```powershell
ipconfig
ping 10.15.71.131
ping 10.15.71.132
Get-Service IAS
Get-WinEvent -FilterHashtable @{LogName='Security';Id=6272,6273;StartTime=(Get-Date).AddHours(-2)} -MaxEvents 20 | Format-List TimeCreated,Id,Message
```

También en Visor de eventos: `Registros de Windows → Seguridad → Filtrar registro actual → 6272,6273`.

## Evidencias para informe/video

- Topología GNS3, interfaces VMnet3, direccionamiento Windows.
- Grupos y cuentas AD.
- Clientes RADIUS y directivas NPS.
- Atributos Cisco VSA, `Service-Type NAS-Prompt`, PAP.
- R1 y SW1: `show running-config`, `show aaa servers`, `show ip ssh`.
- `test aaa` niveles 0, 5, 10 y eventos NPS 6272/6273.
- Accesos SSH de administradores con nivel 15.
- `copy running-config startup-config` y su confirmación.

No publicar capturas con contraseñas ni secretos compartidos visibles.
