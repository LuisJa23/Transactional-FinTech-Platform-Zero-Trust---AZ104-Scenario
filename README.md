---

# Transactional FinTech Platform (Zero-Trust Architecture) - Proyecto Integrador AZ-104

Este repositorio documenta el diseño, la configuración y el despliegue de una plataforma FinTech de procesamiento transaccional en Microsoft Azure. El proyecto aborda de manera integral los dominios evaluados en la certificación **Microsoft Certified: Azure Administrator Associate (AZ-104)**, implementando una arquitectura de red Hub-and-Spoke estricta, controles de identidad granulares, y gobierno de recursos automatizado.

## 🎯 Objetivo Arquitectónico

Construir un ecosistema empresarial seguro (Zero-Trust) para una institución financiera simulada. La plataforma aloja un portal web de alto tráfico (Frontend PaaS) conectado a un núcleo de procesamiento interno (Backend IaaS). Todo el tráfico debe ser inspeccionado, las identidades deben seguir el principio de mínimo privilegio (PoLP), y la retención de datos debe estar automatizada y protegida contra borrado accidental o ataques de ransomware.

## 🛠️ Herramientas y Tecnologías

* **Identidad y Acceso:** Microsoft Entra ID, RBAC (Roles Integrados y Personalizados).
* **Gobernanza:** Azure Policy, Management Groups, Resource Locks, Azure Tags.
* **Redes y Seguridad:** Azure Virtual Networks (VNet), VNet Peering, Azure Firewall, Azure Bastion, User Defined Routes (UDR), Private Endpoints, VNet Integration.
* **Cómputo:** Azure App Service (PaaS), Virtual Machine Scale Sets (VMSS - IaaS).
* **Almacenamiento y Recuperación:** Azure Storage Accounts (Lifecycle Management), Azure Recovery Services Vault (Azure Backup).
* **Observabilidad:** Azure Monitor, Log Analytics Workspace, Action Groups.
* **Infraestructura como Código (IaC):** Bicep / Plantillas ARM (Para la estructura base de redes y almacenamiento).

---

## 🏗️ Fases de Implementación (Diseño Técnico)

### Módulo 1: Identidad y Gobernanza

El objetivo de este módulo es asegurar que las tareas administrativas estén estrictamente segregadas y que cualquier recurso desplegado cumpla con las políticas corporativas financieras.

* **Segmentación de Entra ID:**
* Creación de tres grupos de seguridad: `grp-sec-admins` (Administradores de Seguridad), `grp-dev-paas` (Desarrolladores Web), y `grp-ops-iaas` (Operadores de Infraestructura).


* **Unidades Administrativas (Administrative Units - AUs):**
* Implementación de la AU `AU-FinTech-Core`. Los usuarios del núcleo transaccional residen aquí para delegar la gestión de contraseñas exclusivamente al equipo de soporte de Nivel 1, aislando esta tarea del resto del Tenant.


* **Gobernanza Automatizada (Azure Policy):**
* Creación y asignación de una *Iniciativa de Política* a nivel de Suscripción que impone:
1. Restricción de despliegue exclusivamente a la región `East US`.
2. Auditoría y remediación mediante el efecto *Modify* para asegurar las etiquetas corporativas (`CentroDeCostos: FinTech`, `Entorno: Producción`).
3. Denegación explícita (*Deny*) para la creación de Direcciones IP Públicas en cualquier subred fuera de la red Hub.




* **Control de Acceso Basado en Roles (Azure RBAC):**
* Desarrollo de un **Rol Personalizado (Custom Role) en JSON** llamado "Operador de Mantenimiento FinTech". Este rol permite reiniciar instancias del VMSS y vaciar la caché del App Service, pero utiliza bloqueos en la sección `NotActions` para prohibir terminantemente la eliminación de recursos de red o almacenamiento.



### Módulo 2: Redes Virtuales y Enrutamiento

Diseño de la topología Hub-and-Spoke sin enrutamiento transitivo por defecto, forzando la inspección centralizada.

* **Implementación de Topología (Vía IaC):**
* `vnet-hub-eus` (10.0.0.0/16): Alberga la infraestructura de seguridad.
* `vnet-spoke-web-eus` (10.1.0.0/16): Alberga la aplicación pública.
* `vnet-spoke-core-eus` (10.2.0.0/16): Alberga los datos y el procesamiento interno.
* *Peering:* Configuración bilateral entre el Hub y cada Spoke. El tráfico Spoke-a-Spoke directo está bloqueado.


* **Seguridad Perimetral y Acceso Interno:**
* **Azure Firewall:** Desplegado en la `AzureFirewallSubnet` del Hub para inspección profunda de paquetes (DPI) y filtrado de inteligencia de amenazas.
* **Azure Bastion:** Desplegado en la `AzureBastionSubnet` para proporcionar acceso seguro por RDP/SSH al VMSS sin abrir puertos al internet público.


* **Enrutamiento Forzado (UDR):**
* Implementación de Tablas de Rutas asignadas a las subredes de los Spokes. Una ruta `0.0.0.0/0` (Next Hop Type: Virtual Appliance) obliga a que todo el tráfico de salida y el tráfico cruzado entre Spokes pase por la IP privada del Azure Firewall.



### Módulo 3: Cómputo Elástico y Escalabilidad 

Despliegue de los entornos de ejecución para la lógica de negocio, combinando servicios gestionados (PaaS) y máquinas virtuales dinámicas (IaaS).

* **Frontend (PaaS):**
* **Azure App Service:** Hospeda el portal de acceso para los clientes.
* **VNet Integration:** Configurado en el App Service para permitir que el código de la aplicación alcance recursos con IPs privadas dentro de la red virtual.


* **Backend de Procesamiento (IaaS):**
* **Virtual Machine Scale Set (VMSS):** Desplegado en el `vnet-spoke-core-eus` para procesar las transacciones asíncronas de la base de datos.
* **Autoescalado:** Reglas configuradas para escalar horizontalmente (Scale Out) si el uso medio de CPU supera el 75% durante 10 minutos, y reducirse (Scale In) si cae por debajo del 30%, optimizando los costos de la plataforma FinTech.



### Módulo 4: Almacenamiento y Protección de Datos

Asegurar que los datos financieros en reposo cumplan con los estándares de retención y estén aislados del internet público.

* **Aislamiento de Red (Private Endpoints):**
* La Cuenta de Almacenamiento (Storage Account) tiene bloqueado el acceso a la red pública.
* El acceso se realiza a través de un **Private Endpoint** desplegado en la subred del Core. El App Service y el VMSS resuelven la IP privada interna usando una Zona DNS Privada de Azure integrada con la VNet.


* **Gestión del Ciclo de Vida (Lifecycle Management):**
* Implementación de políticas para mover blobs de registros transaccionales al nivel `Cool` tras 30 días sin modificaciones, y al nivel `Archive` tras 90 días, reduciendo significativamente la factura mensual.


* **Continuidad de Negocio:**
* Despliegue de un **Recovery Services Vault** en el Hub.
* Creación de políticas de Azure Backup para asegurar el estado de las instancias del VMSS y respaldar los recursos compartidos de Azure Files diariamente.
* Aplicación de un **Resource Lock (CanNotDelete)** a nivel del Storage Account para prevenir el borrado accidental por parte de administradores globales.



### Módulo 5: Observabilidad y Respuesta a Incidentes

Centralización de la telemetría para proporcionar visibilidad operativa y respuestas automáticas.

* **Centralización de Logs:**
* Creación de un **Log Analytics Workspace**. Configuración de *Diagnostic Settings* en el Azure Firewall, el App Service y el Recovery Services Vault para enviar registros de actividad y métricas a este espacio único.


* **Reglas de Alertas Proactivas:**
* Configuración de consultas KQL (Kusto Query Language) en Azure Monitor para disparar alertas si se detectan más de 50 conexiones denegadas por el Firewall en un intervalo de 5 minutos, o si la capacidad consumida del Storage Account supera el 85%.
* Integración con un **Action Group** para notificar inmediatamente por correo electrónico a `grp-ops-iaas` y disparar un webhook hacia un sistema de ticketing.



---

---
