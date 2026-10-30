# Incidencias reales y recuperación del laboratorio

## 1. Conectividad de Windows a Cisco

**Síntoma:** R1 y SW1 alcanzables entre sí, pero no podían comunicarse con Windows `10.15.71.130`. Windows tenía la IP correcta en Ethernet1.

**Causa detectada:** Cloud1 estaba conectada a una interfaz de GNS3 VM distinta de la asociada a VMnet3. La GNS3 VM tenía **Network Adapter 4 = VMnet3**; dentro de Linux se expuso como **`eth3`**.

**Solución:** Cloud1 `eth3` ↔ SW1 `Gi0/3`, ambos extremos conectados a VMnet3, Windows Ethernet1 `10.15.71.130/29`. Se recuperó el ping. No fue necesario cambiar NAT ni los roles de Windows.

## 2. Primera autenticación rechazada por RADIUS

La primera prueba desde R1 con `radius0` devolvió `User rejected`. El Visor de eventos NPS mostró 6273 / motivo 16 (credenciales) y `Get-ADUser radius0` reveló `Enabled: False`. Al intentar habilitar la cuenta apareció `ADPasswordComplexityException`.

**Corrección:** restablecer contraseña con política válida, habilitar `radius0`, desactivar cambio obligatorio al primer inicio y volver a probar. Después se obtuvo `User successfully authenticated` y `priv-lvl 0`.

## 3. Dos sintaxis Cisco RADIUS diferentes

- En **R1 Cisco 7200**, `radius server NPS-R1` con `address ipv4` fue rechazado; se utilizó la sintaxis heredada `radius-server host 10.15.71.130 auth-port 1812 acct-port 1813 key ...`. La advertencia de deprecación **no** es un error.
- En **SW1 IOSvL2**, `radius-server host` fue rechazado; funcionó `radius server NPS-SW1`, `address ipv4`, `key`.

Ambos se registraron como clientes separados en NPS, con origen `Fa0/0` y `Vlan20` respectivamente.

## 4. SSH y privilegios

El switch mostró `SSH Disabled` hasta crear claves RSA 2048. Windows OpenSSH rechazó primero algoritmos de intercambio de claves, luego MAC; para R1 también cifrado CBC. Se usaron opciones solo por conexión SSH. El ingreso inicial a SW1 como `oliver` devolvió nivel 1 y el `enable` fue rechazado; al configurar `aaa authorization exec VTY-AUTHZ local` y `authorization exec VTY-AUTHZ` en VTY, el usuario local obtuvo privilegio 15.

Una sesión iniciada con `radius10` no tiene permisos equivalentes a un administrador de nivel 15; los comandos `show running-config` y `configure terminal` pueden ser rechazados. Para documentar running-config usar el SSH de `adminlocal` y `oliver`.

## 5. Apagón y recuperación

Una interrupción eléctrica obligó a reconstruir Cisco y la topología. Las configuraciones modificadas solo en RAM se pierden cuando no han sido copiadas a startup-config. Después de completar/verificar cada equipo, ejecutar:

```cisco
copy running-config startup-config
show startup-config
```

Si Windows avisa `REMOTE HOST IDENTIFICATION HAS CHANGED` tras recrear las claves RSA, **verificar físicamente el nuevo equipo/huella antes de aceptar** y retirar solo la entrada SSH anterior con `ssh-keygen -R IP`.

## 6. Problema al pegar comandos

La consola inicial de GNS3 unía líneas y mostraba comandos concatenados. Se instaló Tera Term y se configuró una pausa de pegado **1000 ms por línea** en `Setup / Additional settings / Copy and Paste`. Los comandos interactivos como `crypto key generate rsa modulus 2048` se ejecutaron por separado.

## 7. Guardado seguro de información

- GitHub público: solo running-configs **sanitizados**.
- Backup privado: `show running-config` completo, `startup-config` y XML exportado de NPS.
- No publicar hashes de usuario/enable, contraseñas, secreto RADIUS ni datos personales de eventos.
- Los extractos recibidos por chat se cortaron al final; no inventar líneas.
