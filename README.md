[README.md](https://github.com/user-attachments/files/32449643/README.md)
# Laboratorio: Dominio Active Directory en Azure con políticas de seguridad (GPO)

![Azure](https://img.shields.io/badge/Azure-Chile%20Central-0078D4)
![Windows Server](https://img.shields.io/badge/Windows%20Server-AD%20DS-005A9E)
![Nivel](https://img.shields.io/badge/Nivel-Laboratorio-lightgrey)

Montaje de un dominio corporativo desde cero sobre Azure: una VM con Windows Server promovida a controlador de dominio (`lab.local`), un cliente unido al dominio y un set de GPO de endurecimiento aplicado y validado.

El objetivo no fue solo "que funcione", sino dejar documentado el **cómo se comprueba** cada paso, que es la parte que normalmente falta en los tutoriales.

---

## Tabla de contenido

- [Arquitectura](#arquitectura)
- [Requisitos previos](#requisitos-previos)
- [Estructura del repositorio](#estructura-del-repositorio)
- [1. Infraestructura en Azure](#1-infraestructura-en-azure)
- [2. Instalación y promoción de Active Directory](#2-instalación-y-promoción-de-active-directory)
- [3. Unión de equipos al dominio](#3-unión-de-equipos-al-dominio)
- [4. Políticas de grupo (GPO)](#4-políticas-de-grupo-gpo)
- [5. Escenarios de validación](#5-escenarios-de-validación)
- [Problemas encontrados y cómo se resolvieron](#problemas-encontrados-y-cómo-se-resolvieron)
- [Consideraciones de seguridad del laboratorio](#consideraciones-de-seguridad-del-laboratorio)
- [Próximos pasos](#próximos-pasos)
- [Referencias](#referencias)

---

## Arquitectura

```
                Internet
                   │
                   │  RDP/3389 (restringido a IP pública propia)
                   ▼
        ┌──────────────────────────────┐
        │   Azure — Chile Central      │
        │ VNet: vnet-lab (10.0.0.0/16) │
        │                              │
        │  ┌────────────────────────┐  │
        │  │ PC-1  (10.0.0.4)       │  │
        │  │ Windows Server         │  │
        │  │ AD DS + DNS            │  │
        │  │ Dominio: lab.local     │  │
        │  └───────────┬────────────┘  │
        │       LDAP/Kerberos/DNS      │
        │  ┌───────────▼────────────┐  │
        │  │ PC-TI (10.0.0.5)       │  │
        │  │Cliente unido al dominio│  │
        │  └────────────────────────┘  │
        └──────────────────────────────┘
```

| Recurso | Nombre | Rol | Notas |
|---|---|---|---|
| Resource Group | `rg-lab-ad` | Contenedor | Región Chile Central |
| VM | `PC-1` | Controlador de dominio | AD DS + DNS, IP privada estática |
| VM / equipo | `PC-TI` | Cliente | Unido a `lab.local` |
| NSG | `nsg-lab` | Filtrado | Entrada 3389 limitada a IP de origen |
| Dominio | `lab.local` | Bosque nuevo | NetBIOS: `LAB` |

> **Nota sobre `.local`:** se usó por tratarse de un laboratorio. En producción conviene un subdominio de un dominio propio (`corp.midominio.com`) para evitar conflictos con mDNS/Bonjour y poder emitir certificados públicos.

📷 *Captura: recursos desplegados en el portal de Azure* — (img/01-recursos-azure.png)

---

## Requisitos previos

- Suscripción de Azure con permisos de creación de recursos.
- Azure CLI (`az`) o acceso al portal.
- Cliente RDP.
- Conocimiento básico de Windows Server y consola de PowerShell.

---

## Estructura del repositorio

```
.
├── README.md
├── docs/
│   └── guia-replicacion.md        # mini-guía paso a paso para replicar el lab
├── img/                            # capturas de cada paso
│   ├── 01-recursos-azure.png
│   ├── 02-nsg-rdp.png
│   ├── 03-instalacion-adds.png
│   ├── 04-promocion-dc.png
│   ├── 05-aduc-contenedores.png
│   ├── 06-usuario-domain-admins.png
│   ├── 07-union-dominio.png
│   ├── 08-aduc-computers.png
│   ├── 09-gpo-firewall.png
│   ├── 10-gpresult.png
│   └── 11-auditoria-eventos.png
└── scripts/
    ├── 01-azure-deploy.sh
    ├── 02-install-adds.ps1
    ├── 03-join-domain.ps1
    ├── 04-gpo-firewall.ps1
    └── 05-validaciones.ps1
```

---

## 1. Infraestructura en Azure

Despliegue del grupo de recursos, la red y la VM que será controlador de dominio.

```bash
# Variables
RG="rg-lab-ad"
LOC="chilecentral"
VM="PC-1"
MI_IP=$(curl -s ifconfig.me)

# Grupo de recursos y red
az group create --name $RG --location $LOC

az network vnet create \
  --resource-group $RG --name vnet-lab \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-servers --subnet-prefix 10.0.0.0/24

# VM Windows Server
az vm create \
  --resource-group $RG --name $VM \
  --image Win2022Datacenter \
  --size Standard_B2ms \
  --vnet-name vnet-lab --subnet snet-servers \
  --private-ip-address 10.0.0.4 \
  --public-ip-sku Standard \
  --admin-username azureadmin \
  --admin-password '<ContraseñaFuerte>'
```

### Regla de entrada para RDP — restringida

El acceso RDP se abre **solo a la IP pública propia**, no a Internet:

```bash
az network nsg rule create \
  --resource-group $RG --nsg-name ${VM}NSG \
  --name allow-rdp-mi-ip --priority 300 \
  --source-address-prefixes ${MI_IP}/32 \
  --destination-port-ranges 3389 \
  --access Allow --protocol Tcp --direction Inbound
```

📷 *Captura: regla de NSG con origen restringido* — `img/02-nsg-rdp.png`

### Validación

```bash
az vm get-instance-view -g $RG -n $VM --query "instanceView.statuses[?starts_with(code,'PowerState')].displayStatus" -o tsv
az network nsg rule list -g $RG --nsg-name ${VM}NSG -o table
```

Desde el cliente RDP: conexión exitosa a la IP pública, puerto 3389.

---

## 2. Instalación y promoción de Active Directory

### Instalación del rol AD DS

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Get-WindowsFeature -Name AD-Domain-Services   # debe figurar como Installed
```

📷 *Captura: rol AD DS instalado* — `img/03-instalacion-adds.png`

### Promoción a controlador de dominio (bosque nuevo)

```powershell
Import-Module ADDSDeployment

Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -DatabasePath "C:\Windows\NTDS" `
  -LogPath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "Contraseña DSRM") `
  -NoRebootOnCompletion:$false `
  -Force:$true
```

El servidor se reinicia al terminar. Tras el reinicio, el inicio de sesión ya es `LAB\azureadmin`.

📷 *Captura: promoción completada* — `img/04-promocion-dc.png`

### Validación del controlador de dominio

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode, PDCEmulator
Get-ADDomainController | Select-Object Name, IPv4Address, IsGlobalCatalog, Site
Get-Service NTDS, DNS, Netlogon, KDC | Select-Object Name, Status

# Registros SRV y localización del DC
nltest /dsgetdc:lab.local
Resolve-DnsName -Name _ldap._tcp.dc._msdcs.lab.local -Type SRV

# Salud general
dcdiag /q
```

📷 *Captura: contenedores y grupos por defecto en ADUC* — `img/05-aduc-contenedores.png`

### Cuenta administrativa de dominio

```powershell
# OU dedicada para cuentas administrativas
New-ADOrganizationalUnit -Name "Administracion" -Path "DC=lab,DC=local"

New-ADUser `
  -Name "adm.israel" `
  -SamAccountName "adm.israel" `
  -UserPrincipalName "adm.israel@lab.local" `
  -Path "OU=Administracion,DC=lab,DC=local" `
  -AccountPassword (Read-Host -AsSecureString "Contraseña") `
  -Enabled $true `
  -ChangePasswordAtLogon $false

Add-ADGroupMember -Identity "Domain Admins" -Members "adm.israel"

# Comprobación
Get-ADGroupMember -Identity "Domain Admins" | Select-Object Name, SamAccountName
```

> **Buena práctica:** separar la cuenta de uso diario de la cuenta con privilegios (modelo de niveles / *tiering*). Nombrar la cuenta administrativa como `Administrador` genera confusión con la cuenta integrada del sistema y complica la auditoría: cuando revisás un evento 4624 querés saber de inmediato de qué cuenta se trata.

📷 *Captura: usuario en Domain Admins* — `img/06-usuario-domain-admins.png`

---

## 3. Unión de equipos al dominio

**Requisito previo que no es opcional:** el cliente debe resolver DNS contra el controlador de dominio. Si apunta a los DNS de Azure o a `8.8.8.8`, la unión falla.

```powershell
# En el cliente (PC-TI) — apuntar DNS al DC
Get-NetAdapter
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.4
Clear-DnsClientCache

# Comprobar que se localiza el dominio ANTES de unir
nltest /dsgetdc:lab.local
Test-NetConnection 10.0.0.4 -Port 389
```

Unión propiamente dicha:

```powershell
Add-Computer -DomainName "lab.local" -Credential (Get-Credential "LAB\adm.israel") -Restart
```

📷 *Captura: equipo unido al dominio* — `img/07-union-dominio.png`

### Validación desde CMD

```cmd
systeminfo | findstr /i "Dominio Domain"
echo %USERDOMAIN%
whoami
nltest /sc_verify:lab.local
```

Y desde el DC, comprobando que el objeto existe en el directorio:

```powershell
Get-ADComputer -Filter * -Properties OperatingSystem, LastLogonDate |
  Select-Object Name, OperatingSystem, DistinguishedName, LastLogonDate
```

📷 *Captura: objeto del equipo en el contenedor Computers de ADUC* — `img/08-aduc-computers.png`

> Los equipos aparecen por defecto en el contenedor `CN=Computers`, que **no** es una OU y no admite GPO vinculadas. Para aplicarles políticas conviene moverlos a una OU propia o redirigir el contenedor por defecto con `redircmp`:
> ```cmd
> redircmp "OU=Equipos,DC=lab,DC=local"
> ```

---

## 4. Políticas de grupo (GPO)

### Creación y vinculación

```powershell
New-ADOrganizationalUnit -Name "Equipos" -Path "DC=lab,DC=local"

New-GPO -Name "SEC - Firewall siempre activo" -Comment "Impide desactivar el firewall de Windows" |
  New-GPLink -Target "OU=Equipos,DC=lab,DC=local" -LinkEnabled Yes
```

### Configuración aplicada

Ruta en **Group Policy Management Editor**:

```
Configuración del equipo
 └ Directivas
   └ Configuración de Windows
     └ Configuración de seguridad
       └ Firewall de Windows Defender con seguridad avanzada
         └ Perfiles: Dominio / Privado / Público
             Estado del firewall ........... Activado (recomendado)
             Conexiones entrantes .......... Bloquear (predeterminado)
             Conexiones salientes .......... Permitir (predeterminado)
```

Complementariamente, para bloquear la desactivación desde el propio equipo:

```
Configuración del equipo
 └ Directivas
   └ Plantillas administrativas
     └ Red → Conexiones de red → Firewall de Windows Defender → Perfil de dominio
         "Proteger todas las conexiones de red" ......... Habilitada
```

La política se extendió a los perfiles **Privado** y **Público**: un portátil que sale de la oficina cambia de perfil de red, y si solo se endurece el perfil de dominio queda desprotegido justo donde más expuesto está.

📷 *Captura: GPO en Group Policy Management* — `img/09-gpo-firewall.png`

### Validación en el cliente

```cmd
gpupdate /force
gpresult /r /scope:computer
gpresult /h C:\temp\gpresult.html & start C:\temp\gpresult.html
```

```powershell
# Estado real del firewall tras aplicar la GPO
Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
netsh advfirewall show allprofiles state

# Intento de desactivación: debe fallar o revertirse en el siguiente refresco
Set-NetFirewallProfile -Profile Domain -Enabled False   # → acceso denegado / sin efecto
```

📷 *Captura: `gpresult /r` mostrando la GPO aplicada* — `img/10-gpresult.png`

> El refresco automático de GPO en clientes ocurre cada 90 minutos (+ desfase aleatorio de hasta 30). En el laboratorio se fuerza con `gpupdate /force`, pero hay que recordar que algunas configuraciones solo se aplican tras reinicio o nuevo inicio de sesión.

---

## 5. Escenarios de validación

### 5.1 Política de contraseñas y bloqueo de cuentas

Aplicada sobre la **Default Domain Policy** (las políticas de contraseña de dominio solo surten efecto a nivel de dominio; para excepciones se usan *Fine-Grained Password Policies*).

| Parámetro | Valor |
|---|---|
| Longitud mínima | 14 caracteres |
| Complejidad | Habilitada |
| Vigencia máxima | 365 días |
| Historial | 24 contraseñas |
| Umbral de bloqueo | 5 intentos |
| Duración del bloqueo | 15 minutos |
| Restablecer contador | 15 minutos |

```powershell
Get-ADDefaultDomainPasswordPolicy
# Prueba: 5 intentos fallidos → cuenta bloqueada
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName, LastLogonDate
```

### 5.2 Auditoría de inicio y cierre de sesión

```
Configuración del equipo → Directivas → Configuración de Windows → Configuración de seguridad
 └ Configuración de directiva de auditoría avanzada
   └ Inicio y cierre de sesión
       Auditar inicio de sesión ............. Correcto y erróneo
       Auditar cierre de sesión ............. Correcto
       Auditar bloqueo de cuenta ............ Correcto y erróneo
```

Eventos a vigilar (base de cualquier caso de uso de SOC):

| Event ID | Significado |
|---|---|
| 4624 | Inicio de sesión correcto |
| 4625 | Inicio de sesión fallido |
| 4634 / 4647 | Cierre de sesión |
| 4740 | Cuenta bloqueada |
| 4768 / 4771 | Kerberos: TGT solicitado / preautenticación fallida |
| 4720 / 4728 | Cuenta creada / agregada a grupo privilegiado |

```powershell
# Inicios de sesión fallidos de las últimas 24 h
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddDays(-1)} |
  Select-Object TimeCreated, @{n='Cuenta';e={$_.Properties[5].Value}}, @{n='Origen';e={$_.Properties[19].Value}}
```

📷 *Captura: eventos de auditoría en el visor de eventos* — `img/11-auditoria-eventos.png`

### 5.3 Restricción de software (AppLocker) — propuesto

```powershell
# Requiere el servicio Application Identity en ejecución
Set-Service -Name AppIDSvc -StartupType Automatic
Get-AppLockerPolicy -Effective | Format-List
```

Enfoque previsto: reglas predeterminadas (`Program Files`, `Windows`) en modo **Auditoría** antes de pasar a **Aplicar**, para no dejar a nadie fuera por una regla mal calibrada.

### 5.4 Bloqueo de USB y BitLocker — siguiente iteración

```
Configuración del equipo → Plantillas administrativas → Sistema
 └ Acceso de almacenamiento extraíble
     "Discos extraíbles: denegar acceso de escritura" ... Habilitada
```

BitLocker con almacenamiento de claves de recuperación en AD DS (`Store BitLocker recovery information in Active Directory Domain Services`).

---

## Problemas encontrados y cómo se resolvieron

| Síntoma | Causa | Solución |
|---|---|---|
| "No se pudo contactar con un controlador de dominio" al unir el equipo | El cliente resolvía DNS contra los servidores de Azure, no contra el DC | Apuntar el DNS del adaptador a `10.0.0.4` y limpiar caché con `Clear-DnsClientCache` |
| La IP del DC cambia tras reiniciar la VM | Asignación dinámica de IP privada | Fijar la IP privada como estática en la configuración de red de Azure |
| Intentos de autenticación fallidos contra RDP | Puerto 3389 abierto a `Internet` en el NSG | Restringir el origen a la IP pública propia; evaluar Azure Bastion o JIT Access |
| La GPO no se aplicaba a los equipos | Los objetos estaban en `CN=Computers`, que no acepta vínculos de GPO | Mover los equipos a una OU y vincular allí, o usar `redircmp` |
| `gpresult /r` mostraba la GPO como "Filtrado: Denegado" | Filtrado de seguridad sin `Authenticated Users` tras los cambios de MS16-072 | Agregar `Authenticated Users` con permiso de *Lectura* en la delegación de la GPO |

---

## Consideraciones de seguridad del laboratorio

Este entorno es deliberadamente simple y **no representa una configuración de producción**. Lo que aquí se aceptó y no debería replicarse tal cual:

- **RDP expuesto a Internet**, aunque sea a una sola IP. Alternativas: Azure Bastion, acceso Just-in-Time de Defender for Cloud, o VPN punto a sitio.
- **Un único controlador de dominio**: sin redundancia, el dominio completo depende de una VM.
- **Dominio `.local`**: aceptable en laboratorio, problemático en entornos reales.
- **Credenciales en texto plano en los comandos de ejemplo**: en un despliegue real irían en Azure Key Vault o pasadas como `SecureString`.
- **Sin copia de seguridad del estado del sistema** del DC ni plan de recuperación del directorio.

---

## Próximos pasos

- [ ] Bloqueo de dispositivos USB por GPO y validación con medios reales
- [ ] BitLocker con custodia de claves de recuperación en AD DS
- [ ] AppLocker en modo auditoría → aplicación
- [ ] Segundo controlador de dominio y prueba de replicación (`repadmin /replsummary`)
- [ ] Envío de eventos de seguridad a un SIEM (Microsoft Sentinel / Wazuh) y creación de reglas de detección
- [ ] Simulación de ataques básicos (password spraying, enumeración con BloodHound) en entorno aislado

---

## Referencias

- [Install Active Directory Domain Services — Microsoft Learn](https://learn.microsoft.com/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services)
- [Group Policy Management — Microsoft Learn](https://learn.microsoft.com/windows-server/identity/ad-ds/manage/group-policy/group-policy-overview)
- [Windows Defender Firewall — Configuración por directiva de grupo](https://learn.microsoft.com/windows/security/operating-system-security/network-security/windows-firewall/)
- [Best Practices for Securing Active Directory](https://learn.microsoft.com/windows-server/identity/ad-ds/plan/security-best-practices/best-practices-for-securing-active-directory)
- [CIS Benchmarks — Microsoft Windows Server](https://www.cisecurity.org/benchmark/microsoft_windows_server)

---

## Licencia y alcance

Material con fines educativos. Los comandos y políticas aquí documentados deben adaptarse y probarse antes de aplicarse en cualquier entorno productivo.
