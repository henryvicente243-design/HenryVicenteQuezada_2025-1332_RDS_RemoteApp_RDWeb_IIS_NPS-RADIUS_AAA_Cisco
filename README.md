 # Configuración de RDS RemoteApp, RD Web Client e IIS con Autenticación Centralizada NPS-RADIUS y AAA en Router Cisco

**Práctica P5**
**Estudiante:** Henry Vicente Quezada
**Matrícula:** 2025-1332
**Institución:** Instituto Tecnológico de las Américas (ITLA)

## Video de demostración

[Enlace al video en YouTube (no listado)](PEGAR_ENLACE_AQUI)

---

## 1. Objetivo de la red

El objetivo de esta práctica es implementar un entorno de publicación de aplicaciones remotas mediante los servicios de Escritorio Remoto (RDS) de Windows Server, integrando dos métodos de acceso —RD Web Access y RD Web Client— para consumir una aplicación RemoteApp (Microsoft Edge) que despliega una página web personalizada alojada en IIS.

De forma complementaria, se implementa un servidor NPS (Network Policy Server) actuando como servidor RADIUS, encargado de centralizar la autenticación y autorización de acceso administrativo a un router Cisco mediante AAA. El NPS diferencia dos niveles de privilegio (15 y 1) según el grupo de Active Directory al que pertenece el usuario, utilizando el atributo Cisco-AV-Pair para transmitir el nivel de privilegio correspondiente al dispositivo de red.

Con esto se busca demostrar el funcionamiento integral de un esquema de autenticación centralizada (AAA vía RADIUS) aplicado tanto a servicios de aplicaciones remotas como a la administración de infraestructura de red, replicando un escenario típico de entorno empresarial.

## 2. Topología de red

La topología está compuesta por un router (RTR-CORE-LAB) que interconecta dos segmentos de red: uno hacia el cliente Windows y otro hacia el servidor Windows, sin switches intermedios. El direccionamiento IP se derivó de los dígitos de la matrícula del estudiante (13 y 32).

![Topología](capturas/20_Topologia.png)

### 2.1 Direccionamiento IP

| Dispositivo | Interfaz | Dirección IP |
|---|---|---|
| Windows Client | NIC (VMnet6) | 10.13.1.10 /24 |
| Router RTR-CORE-LAB | e0/0 (hacia Client) | 10.13.1.1 /24 |
| Router RTR-CORE-LAB | e0/1 (hacia Server) | 10.32.1.1 /24 |
| Windows Server (SRV-2025-1332) | NIC (VMnet7) | 10.32.1.10 /24 |

No se utilizaron VLANs en esta topología; la segmentación se realiza a nivel de interfaz física/routing directo entre las dos subredes.

## 3. Configuración del Windows Server

### 3.1 Instalación de Active Directory Domain Services

El servidor se promovió a Controlador de Dominio (DC), requisito previo para la instalación de RDS mediante el asistente Quick Start.

| Parámetro | Valor |
|---|---|
| Hostname | SRV-2025-1332 |
| Dominio (FQDN) | henryvicente.local |
| NetBIOS | HENRYVICENTE |

### 3.2 Configuración de página personalizada en IIS

Se configuró una página HTML personalizada, identificada con el nombre y matrícula del estudiante, ubicada en `C:\inetpub\wwwroot\index.html`.

![Página IIS](capturas/06_PaginaIIS.png)

### 3.3 Certificado SSL autofirmado

Se generó un certificado SSL autofirmado mediante `New-SelfSignedCertificate` con la extensión de uso "Server Authentication", necesaria para evitar el error `ERR_SSL_KEY_USAGE_INCOMPATIBLE`. Debido a incompatibilidad del cmdlet `Export-PfxCertificate` con el asistente de RDS, la exportación del certificado se realizó con `certutil -exportPFX`.

### 3.4 Instalación de RDS y publicación de RemoteApp

Se instalaron los roles de Session Host, Connection Broker y Web Access mediante el asistente Quick Start de RDS. Se publicó como RemoteApp la aplicación Microsoft Edge, configurada con el argumento `http://10.32.1.10` para que abra automáticamente la página personalizada de IIS al ejecutarse.

![RemoteApp publicando](capturas/01_RemoteAppPublicando.png)

### 3.5 Servicio RD Web Access

Disponible en `https://10.32.1.10/RDWeb`, donde el usuario se autentica con sus credenciales de dominio para visualizar las aplicaciones publicadas.

![Login RD Web Access](capturas/02_RDWebAccess-Login.png)
![Logueado RD Web Access](capturas/03_RDWebAccess-Logueado.png)
![RemoteApp vía Web Access](capturas/07_RemoteApp-MostrandoPaginaIIS_WebAccess.png)

### 3.6 Servicio RD Web Client

Versión HTML5, disponible en `https://10.32.1.10/RDWeb/webclient`. Su puesta en marcha requirió actualizar el módulo `PowerShellGet` e importar el certificado del broker con `Import-RDWebClientBrokerCert`.

![Login RD Web Client](capturas/04_RDWebClient-Login.png)
![Logueado RD Web Client](capturas/05_RDWebClient-Logueado.png)
![RemoteApp vía Web Client](capturas/08_RemoteApp-MostrandoPaginaIIS_WebClient.png)

### 3.7 Configuración de NPS (RADIUS Server)

Se configuró el rol NPS actuando como servidor RADIUS. El router fue registrado como cliente RADIUS con la IP 10.32.1.1 (interfaz e0/1).

**Grupos de Active Directory por nivel de acceso:**
- `NivelAcceso15` — privilegio administrativo completo (privilege 15)
- `NivelAcceso1` — privilegio de solo consulta (privilege 1)

![Grupo Nivel 15](capturas/09_GrupoNivel15.png)
![Grupo Nivel 1](capturas/11_GrupoNivel1.png)

**Usuarios de prueba:**
- `adminradius` — miembro de `NivelAcceso15`
- `userradius` — miembro de `NivelAcceso1`
- Contraseña (ambos): `R4dius#2025`

**Políticas de red NPS y atributo Cisco-AV-Pair:**

Se configuraron dos políticas de red, una por grupo, devolviendo el atributo `Cisco-AV-Pair` con el nivel de privilegio correspondiente. Fue necesario ajustar `Service-Type` a "NAS Prompt" y eliminar `Framed-Protocol PPP`, ya que su presencia generaba el error *"This line may not run PPP"* durante el intento de conexión SSH.

![Política Nivel 15](capturas/10_PoliticaNivel15-AVPair.png)
![Política Nivel 1](capturas/12_PoliticaNivel1-AVPair.png)

## 4. Configuración del Router

### 4.1 Usuario local y contraseñas de acceso

Usuario local con privilegio 15 como respaldo (fallback) ante indisponibilidad del servidor RADIUS, más la contraseña de modo privilegiado:

```
username admin privilege 15 secret Cisco123!
enable secret Cisco123!
```

![Usuario local](capturas/13_UsuarioLocal.png)

### 4.2 Configuración de AAA para autenticación vía RADIUS

```
aaa new-model
aaa authentication login default group radius local
aaa authorization exec default group radius local
radius server NPS-SERVER
 address ipv4 10.32.1.10 auth-port 1812 acct-port 1813
 key RadiusKey123!
```

> **Nota técnica:** se descartó `aaa authentication enable default group radius enable`, ya que no era requisito del enunciado y generaba fallos por el usuario especial `$enab15$` que RADIUS espera para autenticar el modo enable de forma independiente.

![Configuración AAA](capturas/14_AAA-Config.png)

### 4.3 Logs de autenticación, autorización y RADIUS

```
terminal monitor
debug aaa authentication
debug aaa authorization
debug radius
```

![Debugs AAA/RADIUS](capturas/15_Debugs-Authentication-Authorization-Radius.png)

## 5. Pruebas y validación desde el cliente

### 5.1 Conectividad

![Ping cliente-servidor](capturas/21_Ping_de_windowsClient.png)

### 5.2 SSH vía RADIUS — usuario nivel 15

```
ssh -l adminradius 10.13.1.1
Password: R4dius#2025
```

El usuario ingresó directamente en modo privilegiado (`RTR-CORE-LAB#`), confirmado con `show privilege` (nivel 15).

![SSH nivel 15](capturas/18_SSH-Nivel15.png)

### 5.3 SSH vía RADIUS — usuario nivel 1

```
ssh -l userradius 10.13.1.1
Password: R4dius#2025
```

El usuario ingresó en modo usuario (`RTR-CORE-LAB>`), confirmado con `show privilege` (nivel 1).

![SSH nivel 1](capturas/19_SSH-Nivel1.png)

### 5.4 Verificación final del servidor RADIUS y sesiones activas

```
undebug all
show aaa servers
```

Confirmó que el servidor RADIUS estaba activo (`State: current UP`), con 5 solicitudes enviadas, 3 aceptadas y 2 rechazadas durante las pruebas.

![show aaa servers](capturas/16_ShowAAAServers.png)

```
show aaa sessions
```

Mostró la sesión activa del usuario `adminradius`, autenticado desde 10.13.1.10.

![show aaa sessions](capturas/17_ShowAAASessions.png)

## 6. Incidencias resueltas durante el proyecto

- El asistente Quick Start de RDS requería un dominio activo → se promovió el servidor a Controlador de Dominio.
- Error `ERR_SSL_KEY_USAGE_INCOMPATIBLE` en el certificado SSL → resuelto regenerándolo con la extensión Server Authentication.
- Incompatibilidad de `Export-PfxCertificate` con el asistente de RDS → resuelto con `certutil -exportPFX`.
- RD Web Client sin acceso a internet → resuelto ajustando DNS/NAT del servidor.
- Certificado del broker faltante para RD Web Client → resuelto con `Import-RDWebClientBrokerCert`.
- Contraseña de AD rechazada por contener el nombre de usuario → ajustada conforme a políticas de complejidad.
- Error *"This line may not run PPP"* en SSH → corregido `Service-Type` y eliminado `Framed-Protocol PPP` en la política NPS.
- Ruta por defecto duplicada/obsoleta en el Windows Server → eliminada con `route delete`.

## 7. Conclusiones

La práctica permitió integrar de forma funcional dos frentes de la infraestructura de TI: la publicación de aplicaciones remotas mediante RDS (RD Web Access y RD Web Client) y la administración segura de un dispositivo de red mediante autenticación centralizada AAA/RADIUS. Se comprobó que el atributo Cisco-AV-Pair, distribuido desde políticas de NPS diferenciadas por grupo de Active Directory, permite asignar de forma dinámica distintos niveles de privilegio (15 y 1) a los usuarios que acceden por SSH al router, sin necesidad de mantener credenciales ni permisos de forma local en cada dispositivo.

Asimismo, el proceso evidenció la importancia de resolver dependencias de infraestructura (promoción a Controlador de Dominio, gestión de certificados SSL, corrección de atributos RADIUS específicos de Cisco) para lograr un entorno funcional de extremo a extremo, replicando condiciones similares a las de un entorno de producción empresarial.

