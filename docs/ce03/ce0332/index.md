# CE0332-CE0333 - S2 - E3: Implementación, Gestión y Control de Centro de Datos

**UNIVERSIDAD PERUANA UNIÓN**

Facultad de Ingeniería y Arquitectura · Escuela Profesional de Ingeniería de Sistemas

**SEMESTRE 2 · ENTREGABLE 3 DE 3**

**Implementación, Gestión y Control de Centro de Datos**

*(CE0332–CE0333 · Línea de Infraestructura — CE03)*

Centro de datos híbrido del ERP «Articulos» — Salud Total SAC

| **Campo**            | **Detalle**                                                                                     |
|----------------------|-------------------------------------------------------------------------------------------------|
| Responsables         | Jhosef Anthony Quispe Huarilloclla · Jose Samuel Turpo Cauna · Yesenia Alejandra Gutierrez Anco |
| Versión y fecha      | v2.0 — 19 de noviembre de 2026                                                                  |
| Diseño de referencia | CE0331 E2 v2.0 (Análisis y Diseño de Centro de Datos Híbrido)                                   |

## Control de versiones

| **Versión** | **Fecha**  | **Cambios**                                                                                                                                                                          |
|-------------|------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1.0         | 30/06/2026 | Servicios de centro de datos en laboratorio (DNS, LDAP, NTP, LVM, respaldo cifrado) sobre el diseño anterior (con Kafka, vCenter y SAN)                                              |
| 2.0         | 19/11/2026 | Rehecho según la plantilla S2-E3 y alineado al E2 v2.0: trazabilidad, despliegue real en AWS, accesos, inventario, registro de recuperación, monitoreo, control de cambios y brechas |

Anexos: Anexos/Registro_Recuperacion_Inventario_S2E3.xlsx (registro de recuperación, trazabilidad, inventario, accesos y brechas) y version_anterior_v1/Evidencias_DataCenter_SaludTotal.zip (configuraciones y salidas de junio).

## 1. Resumen ejecutivo de la implementación del centro de datos

Proyecto: centro de datos híbrido del ERP «Articulos» de Salud Total SAC, diseñado en el E2 v2.0 (host ESXi de contingencia y seguridad en Juliaca y una EC2 con Docker Compose en AWS us-east-2).

Lo desplegado. En AWS: una EC2 Ubuntu 24.04 con Elastic IP en la VPC por defecto, Security Group con SSH para el administrador y el ERP en el puerto 8090, y los cinco contenedores del ERP en estado saludable. On-premise: el host ESXi no se desplegó; en junio se ejecutaron en una VM Linux de laboratorio los servicios de soporte DNS y LDAP, y un respaldo cifrado con restauración verificada.

Identidades y accesos: SSH solo con llave y desde la IP del administrador, llaves JWT fuera de git y rol de base de datos sin superusuario están implementados; MFA, usuario IAM y rol de instancia no tienen evidencia.

Respaldo y recuperación: se ejecutaron tres recuperaciones con el mismo stack del ERP: rollback de un despliegue fallido en 18,5 s (RTO objetivo 4 h), restauración de la base en ≈ 4,2 s con conteos idénticos, y restauración de un respaldo cifrado en junio. La recuperación de una VM, de un snapshot de AWS y de la VPN no se ejecutaron.

Conclusión: la parte en la nube cumple el diseño en lo esencial (una EC2 con el ERP en contenedores, filtrado perimetral en el Security Group, recuperación probada de servicio y datos), con dos desvíos justificados (VPC por defecto y puerto 8090 sin TLS). La parte on-premise no se implementó; las brechas y su plan de cierre están en la sección 10.

## 2. Trazabilidad con el diseño de centro de datos

| **Elemento diseñado**                                        | **¿Según diseño?** | **Cambio / justificación**                                                                                                                                                |
|--------------------------------------------------------------|--------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Host ESX-JUL-01, vSwitches, port groups y 7 VM (E2 §5.2)     | No                 | No se dispuso de servidor ni de licencia; la equivalencia técnica (Workstation o VirtualBox) tampoco se montó. Queda como brecha B01                                      |
| Servicios de soporte on-premise (DNS, directorio)            | Sí, en laboratorio | DNS (dnsmasq) y directorio LDAP ejecutados en una VM Linux de laboratorio en junio; no forman parte del diseño v2.0 pero sirven como evidencia de operación               |
| NAS de respaldos y RAID 1                                    | Parcial            | Se ejecutó la creación de PV/VG de LVM sobre dispositivos de bucle; el RAID se documentó sin instanciarlo (el kernel del laboratorio no cargaba md)                       |
| VPC-ST 10.50.0.0/16 con subredes por capa (E2 §6.2)          | Con cambio         | Se utilizó la VPC por defecto 172.31.0.0/16 en el despliegue inicial As-Is; la migración a la VPC-ST dedicada (10.50.0.0/16) forma parte del plan de endurecimiento (B02) |
| EC2-ERP-01 t3.large con Docker Compose (E2 §6.3)             | Sí, con ajuste     | EC2 Ubuntu 24.04 con Elastic IP y los cinco contenedores del ERP saludables; el tipo de instancia no está registrado                                                      |
| sg-erp y NACL (E2 §6.4)                                      | Parcial            | Security Group con 22 (IP del administrador) y 8090; sin NACL dedicada; puerto público 8090 sin TLS en lugar de 443                                                       |
| Rol IAM, S3 de respaldos, DLM, CloudWatch, Budgets (E2 §6.3) | Sin evidencia      | Procedimiento de configuración y monitoreo documentado; capturas en proceso de consolidación operativa                                                                    |
| Respaldo 3-2-1 y restauración (E2 §10)                       | Parcial            | pg_dump y restauración verificados con el mismo stack en local y respaldo cifrado verificado en junio; falta en la EC2                                                    |
| Monitoreo del ERP (Prometheus, 6 alertas)                    | Sí                 | Mismo docker-compose que la EC2; verificado en local                                                                                                                      |

## 3. Implementación del entorno on-premise (VMware ESXi)

No se desplegó el host ESX-JUL-01 ni una equivalencia técnica, por lo que no hay vSwitches, port groups, NIC teaming, VM, firewall virtual ni IDS dentro de un hipervisor que documentar. Lo que se ejecutó on-premise fue en una VM Linux de laboratorio (1 vCPU, 3,9 GiB de RAM; ev03), en junio:

| **Servicio**            | **Configuración**                                                                       | **Resultado**                                             | **Vigencia frente al diseño v2.0**                                                                      |
|-------------------------|-----------------------------------------------------------------------------------------|-----------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| DNS interno (dnsmasq)   | Zona saludtotal.pe                                                                      | Resolución verificada con dig (ev01)                      | Útil como servicio de soporte; los registros de Kafka y vCenter son del diseño anterior y ya no aplican |
| Directorio LDAP (slapd) | dc=saludtotal,dc=pe; unidades Juliaca y Puno; grupos Cajeros, Médicos y Administradores | Consulta verificada; bind sin contraseña rechazado (ev02) | Equivalente a un directorio corporativo; el ERP usa su propia autenticación JWT                         |
| NTP (chrony)            | Configuración                                                                           | Documentado                                               | Vigente                                                                                                 |
| LVM                     | PV y VG sobre dispositivos de bucle                                                     | Creación de PV/VG ejecutada (ev_lvm_pvvg)                 | Ilustra el aprovisionamiento; el diseño v2.0 usa RAID 1 en el host y un NAS                             |
| RAID 6 y SAN iSCSI      | Configuración documentada                                                               | No instanciado (sin módulos md en el laboratorio)         | Obsoleto: el diseño v2.0 no usa SAN ni RAID 6                                                           |

## 4. Implementación del entorno AWS

### 4.1 VPC desplegada

| **Elemento**     | **Real**                                                                       | **Diseño (E2)**                           |
|------------------|--------------------------------------------------------------------------------|-------------------------------------------|
| VPC              | VPC por defecto 172.31.0.0/16                                                  | VPC-ST 10.50.0.0/16                       |
| Subred           | Subred pública por defecto de la zona de la instancia (IP privada 172.31.4.35) | snet-publica-a 10.50.0.0/24               |
| Route table      | Tabla principal de la VPC por defecto con ruta 0.0.0.0/0 al Internet Gateway   | rtb-publica: 0.0.0.0/0 → IGW; sedes → VGW |
| Internet Gateway | El de la VPC por defecto                                                       | IGW-ST                                    |
| NAT              | No hay (no se necesita: la única instancia es pública)                         | No en la fase 1                           |

### 4.2 Instancias EC2 desplegadas

| **Instancia**                           | **Tipo**                                | **IP privada / pública**                           | **Estado**                                                                                  |
|-----------------------------------------|-----------------------------------------|----------------------------------------------------|---------------------------------------------------------------------------------------------|
| EC2 del ERP (Ubuntu 24.04, disco 20 GB) | No registrado (el diseño pide t3.large) | 172.31.4.35 / Elastic IP pública (198.51.100.50)\* | Activa en el despliegue inicial; cinco contenedores saludables; arranque del backend ≈ 35 s |

### 4.3 Security Groups y NACL configurados

| **Nombre**               | **Alcance**        | **Regla efectiva**                                                                                                                                                                                       |
|--------------------------|--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Security Group de la EC2 | Instancia          | 22 desde la IP del administrador; 8090 desde 0.0.0.0/0; el resto cerrado                                                                                                                                 |
| NACL                     | Subred             | NACL por defecto de la VPC (sin reglas adicionales a nivel de subred; el control de acceso efectivo recae en el Security Group de la instancia, quedando la NACL dedicada para la fase de endurecimiento |
| Publicación en el host   | docker-compose.yml | 8085, 8087, 9090 y 5433 publicados en 0.0.0.0 (protegidos solo por el Security Group): no conformidad H02 del SGSI                                                                                       |

## 5. Gestión de identidades y accesos del centro de datos

### 5.1 AWS IAM

| **Control**                                         | **Implementación**                                                    | **Estado**                  |
|-----------------------------------------------------|-----------------------------------------------------------------------|-----------------------------|
| EC2 por SSH                                         | Llave .pem del administrador; Security Group solo desde su IP pública | Implementado (resumen §6.3) |
| Usuario IAM administrador distinto de root, con MFA | —                                                                     | Sin evidencia               |
| MFA en la cuenta raíz                               | —                                                                     | Sin evidencia               |
| Rol de instancia para respaldos en S3               | —                                                                     | No implementado             |

### 5.2 Accesos administrativos (ESXi y servidores)

| **Control**                             | **Implementación**                                                               | **Estado**                                 |
|-----------------------------------------|----------------------------------------------------------------------------------|--------------------------------------------|
| Llaves JWT                              | Generadas en la VM y montadas desde secrets/; fuera de git                       | Implementado; rotación periódica pendiente |
| Base de datos                           | Rol de aplicación st_app sin superusuario (script de inicialización del compose) | Implementado                               |
| Acceso administrativo a ESXi y firewall | —                                                                                | No aplica (no desplegados)                 |

Política aplicada en la práctica: SSH con llave y sin contraseña; las llaves JWT se generan en el servidor y no viajan por git (incidente INC-02 del SGSI); el escaneo de secretos del pipeline bloquea llaves nuevas en el repositorio. La rotación periódica de llaves JWT y SSH no se ha ejecutado todavía.

## 6. Inventario final de activos del centro de datos

Coherente con el inventario del SGSI (activos A01–A15), con la evidencia de cada componente:

| **Activo**                                                       | **Propietario**          | **Ubicación**                   | **Criticidad** | **Evidencia**                                                                                    |
|------------------------------------------------------------------|--------------------------|---------------------------------|----------------|--------------------------------------------------------------------------------------------------|
| Instancia EC2 del ERP (Ubuntu 24.04, Elastic IP 198.51.100.50\*) | Responsable de nube      | AWS us-east-2                   | Crítica        | Despliegue operativo en AWS; procedimiento de validación documentado                             |
| Security Group (22 administrador, 8090 público)                  | Responsable de nube      | AWS                             | Crítica        | Reglas operativas en AWS; procedimiento de validación documentado                                |
| Contenedores frontend, backend, mensajería, db, prometheus       | Administrador de TI      | EC2 (Docker Compose)            | Crítica        | docker compose ps con 5 servicios saludables (resumen) y en local (pipeline_20261119-063850.log) |
| Base clinica_db (PostgreSQL 17, Flyway V1 y V2)                  | Administrador de TI      | Contenedor db                   | Crítica        | Restauración verificada (respaldo_restauracion.txt)                                              |
| Llaves JWT en secrets/jwt                                        | Responsable de seguridad | EC2                             | Crítica        | Incidente INC-02 del SGSI (generadas en la VM)                                                   |
| Llave SSH de la EC2                                              | Administrador de TI      | Equipo del administrador        | Crítica        | Uso en operación (sin captura)                                                                   |
| Prometheus con 6 reglas de alerta                                | Administrador de TI      | Contenedor prometheus           | Media          | monitoring/alertas.yml y figura 2.1 de CE02 E4                                                   |
| Servicios de laboratorio: DNS y LDAP                             | Administrador de TI      | VM Linux de laboratorio (junio) | Media          | version_anterior_v1/Evidencias_DataCenter_SaludTotal.zip (ev01, ev02)                            |
| Host ESXi y sus 7 VM                                             | Administrador de TI      | Rack de Juliaca                 | Alta           | No desplegado (brecha B01)                                                                       |
| NAS de respaldos                                                 | Administrador de TI      | Rack de Juliaca                 | Alta           | No desplegado; LVM de laboratorio (ev_lvm_pvvg)                                                  |

## 7. Gestión de continuidad — respaldo y recuperación

Estrategia de respaldo implementada: pg_dump en formato custom con retención de 7 copias (ci/respaldo.ps1) y verificación por restauración en una base temporal; en junio, respaldo cifrado con AES-256. El snapshot EBS diario, la copia a S3 y la copia en el NAS del diseño (regla 3-2-1) no están implementados en la EC2.

Procedimiento de restauración: (1) copiar el respaldo al contenedor db; (2) crear una base temporal; (3) pg_restore --no-owner --no-acl; (4) comparar conteos de productos, venta_registro, kardex, orden_reabastecimiento y flyway_schema_history; (5) si coinciden, apuntar el servicio a la base restaurada o renombrarla.

| **ID** | **Escenario**                                                             | **RTO obj.** | **RTO obtenido**                              | **RPO obj.** | **Estado**                     |
|--------|---------------------------------------------------------------------------|--------------|-----------------------------------------------|--------------|--------------------------------|
| RC-01  | Recuperación de un servicio: despliegue fallido del backend               | 4 h          | 18,5 s                                        | —            | Ejecutado (local)              |
| RC-02  | Restauración de la base clinica_db desde respaldo                         | 4 h          | ≈ 4,2 s (1,4 s respaldo + 2,8 s restauración) | 24 h         | Ejecutado (local)              |
| RC-03  | Restauración de un respaldo cifrado (AES-256) con 200 002 filas de kardex | 4 h          | No medido                                     | 24 h         | Ejecutado (laboratorio, junio) |
| RC-04  | Recuperación de una VM desde respaldo                                     | 24 h         | —                                             | 24 h         | No ejecutado                   |
| RC-05  | Recuperación de un recurso AWS desde snapshot                             | 4 h          | —                                             | 24 h         | No ejecutado                   |
| RC-06  | Recuperación de la VPN tras una interrupción                              | 1 h          | —                                             | —            | No aplica                      |

## 8. Monitoreo y control operativo del centro de datos

- **ERP y contenedores:** Prometheus con métricas del backend, histogramas de latencia y 6 reglas de alerta (backend caído, 5xx, latencia de ventas, eventos outbox atascados o con error, órdenes de reabastecimiento); Actuator y healthchecks de Docker. Verificado con el mismo stack (CE02 E4, figura 2.1).

- **Recursos del servidor:** en el laboratorio de junio se registraron CPU, memoria, disco y servicios activos (ev03); en la EC2 no hay alarmas de capacidad evidenciadas.

- **Host ESXi:** no aplica (no desplegado).

- **CloudWatch:** sin evidencia; el diseño pide StatusCheckFailed_System con recuperación automática, CPU \> 70 % y disco \> 80 % (B05).

## 9. Control de cambios y gestión de configuración

Repositorio técnico: el código del ERP y su infraestructura están en git (copia articulos-e3 y el repositorio GitHub del equipo citado en el resumen). Contenido relevante:

| **Elemento**                              | **Ubicación**                                                                                                        | **Estado**                                                                    |
|-------------------------------------------|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| README con instrucciones para reconstruir | articulos-e3/README.md (docker compose, .env, iniciar-docker.ps1)                                                    | Disponible                                                                    |
| Infraestructura como código               | docker-compose.yml, Dockerfile ×3, db/init, monitoring/                                                              | Disponible                                                                    |
| Scripts                                   | ci/pipeline.ps1, ci/rollback.ps1, ci/respaldo.ps1                                                                    | Disponible                                                                    |
| Registros                                 | ci/reportes/ (ejecuciones del pipeline), pruebas/evidencias/                                                         | Disponible                                                                    |
| Diagramas                                 | Carpetas Diagramas de los entregables E1 y E2 de CE03                                                                | Disponible                                                                    |
| Secretos                                  | .env y secrets/ excluidos por .gitignore; escaneo de secretos en cada build                                          | Sin secretos nuevos; llaves de desarrollo heredadas registradas (NC-01)       |
| Historial de cambios                      | Historial de git; en el repositorio del equipo se registró y revirtió el borrado accidental de una carpeta duplicada | Los cambios de E3/E4 de la copia articulos-e3 aún no están confirmados en git |

## 10. Resultados y conclusiones

- **Operativo conforme al diseño:** la EC2 con el ERP en contenedores, el perímetro mínimo por Security Group, el monitoreo con alertas y la recuperación de servicio y de datos probada (RC-01 a RC-03).

- **Prueba de recuperación:** rollback en 18,5 s y restauración de la base en ≈ 4,2 s, ambos muy por debajo del RTO de 4 h; ejecutadas con el mismo docker-compose de la EC2, en el entorno local.

- Limitaciones: La validación de la infraestructura física on-premise se realizó mediante servicios de soporte en entorno de laboratorio; en la nube de AWS, las evidencias operativas de VPC, IAM y CloudWatch se consolidan mediante procedimientos de captura y validación documentados.

- **Mejoras propuestas:** cerrar las brechas siguientes, empezando por las de menos esfuerzo (MFA, alarmas y respaldo en la EC2).

| **ID** | **Brecha**           | **Acción**                                                                                                                                                                | **Esfuerzo** |
|--------|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
| B01    | Host ESXi y VM       | Montar ESXi o una equivalencia técnica (Workstation/VirtualBox) con los port groups de VLAN 10, 20, 70 y 100 y al menos APP-CONT-01 y DB-CONT-01; capturar vSwitches y VM | 2 a 3 h      |
| B02    | VPC dedicada         | Migrar la instancia EC2 a la VPC-ST dedicada 10.50.0.0/16 con snet-publica-a aplicando el Security Group sg-erp y la NACL diseñada                                        | 1 a 2 h      |
| B03    | Recuperación en AWS  | Snapshot del volumen, nueva EC2 desde él y medición del tiempo (RC-05)                                                                                                    | 1 h          |
| B04    | IAM y MFA            | MFA en la cuenta raíz, usuario IAM administrador, rol de instancia para S3; capturas                                                                                      | 15 min       |
| B05    | Monitoreo en la nube | Alarmas de CloudWatch (StatusCheck, CPU, disco) y AWS Budgets; capturas                                                                                                   | 30 min       |
| B06    | Respaldo en la EC2   | Programar el pg_dump de las 02:00 a S3 y ejecutar una restauración allí                                                                                                   | 30 min       |

## 11. Referencias

\[1\] VMware by Broadcom, "vSphere Networking" y "vSphere Storage — ESXi 8.0", 2025.

\[2\] Amazon Web Services, "Amazon EC2 User Guide", "IAM User Guide" y "AWS Backup Developer Guide", 2026.

\[3\] ISO/IEC 27001:2022, "Information security management systems — Requirements", ISO, 2022.

\[4\] CIS, "CIS Amazon Web Services Foundations Benchmark", v3.0, 2024.

\[5\] CIS, "CIS VMware ESXi 8.0 Benchmark", 2024.

\[6\] PostgreSQL Global Development Group, "PostgreSQL 17 Documentation — Backup and Restore", 2025.


---

### Archivos del entregable

- [:material-file-word: Descargar documento original (.docx)](Entregable_S2E3_Centro_Datos_SaludTotal.docx)

**Anexos**

- [:material-file-excel: Registro_Recuperacion_Inventario_S2E3.xlsx](archivos/Anexos/Registro_Recuperacion_Inventario_S2E3.xlsx)
