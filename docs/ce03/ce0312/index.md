# CE0312-CE0313 - S2 - E2: Implementación, Testing y Monitoreo de Red

**UNIVERSIDAD PERUANA UNIÓN**

Facultad de Ingeniería y Arquitectura · Escuela Profesional de Ingeniería de Sistemas

**SEMESTRE 2 · ENTREGABLE 2 DE 3**

**Implementación, Testing y Monitoreo de Red**

*(CE0312–CE0313 · Línea de Infraestructura — CE03)*

Red híbrida del ERP «Articulos» — Salud Total SAC (Juliaca, Puno y AWS)

| **Campo**            | **Detalle**                                                                                     |
|----------------------|-------------------------------------------------------------------------------------------------|
| Responsables         | Jhosef Anthony Quispe Huarilloclla · Jose Samuel Turpo Cauna · Yesenia Alejandra Gutierrez Anco |
| Versión y fecha      | v2.0 — 19 de noviembre de 2026                                                                  |
| Diseño de referencia | CE0311 E1 v2.1 (Análisis y Diseño de Red)                                                       |

## Control de versiones

| **Versión** | **Fecha**     | **Cambios**                                                                                                           |
|-------------|---------------|-----------------------------------------------------------------------------------------------------------------------|
| 1.0         | Junio de 2026 | Emulación de la red con network namespaces y pruebas (estaba archivado en la carpeta del SGSI)                        |
| 2.0         | 19/11/2026    | Reorganizado: trazabilidad con el E1 v2.1, perímetro real en AWS, resultados con su validez, brechas y plan de cierre |

Anexos: Anexos/Matriz_Pruebas_Red_S2E2.xlsx (trazabilidad, pruebas de emulación, pruebas en AWS, matriz C01–C20 del E1, validez de la evidencia y brechas) y version_anterior_v1/Evidencias_Red_SaludTotal.zip (configuraciones y salidas originales).

## 1. Resumen ejecutivo

Este entregable documenta lo que se implementó y probó de la red diseñada en el E1 v2.1. Hay dos entornos con evidencia: una emulación de la red de las sedes con network namespaces de Linux (junio de 2026) y el perímetro del ERP en AWS (una EC2 con Docker Compose, Security Group con 22 para el administrador y 8090 público, Nginx como único punto de entrada).

Resultados: de 11 pruebas de la emulación, 9 cumplen —ruteo, aislamiento de invitados y de la clínica con contadores de la ACL, salida de invitados a Internet y lectura SNMP del núcleo— y 2 no son concluyentes porque el retardo y el límite de ancho de banda del enlace de Puno no llegaron a aplicarse. En AWS, el perímetro mínimo está funcionando, pero las pruebas P01–P10 carecen de capturas, salvo las que se repitieron con el mismo stack en local (health UP y 401).

Brechas: la VPN IPsec, el IDS/IPS y el honeypot no se desplegaron; los puertos internos de la EC2 siguen publicados en el host y el servicio usa HTTP. La sección 8 da la acción mínima para cerrar cada una. Las 20 pruebas de conectividad del E1 (C01–C20) quedan pendientes porque requieren VPN, VLAN y firewalls reales.

## 2. Trazabilidad con el diseño de red

| **Elemento del diseño**                                   | **Dónde**                  | **Estado**                 | **Detalle / cambio justificado**                                                                   |
|-----------------------------------------------------------|----------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| VLAN de Juliaca (10, 30, 50, 90) y su direccionamiento    | Emulación (junio)          | Implementado en emulación  | Mismas subredes que el E1 v2.1; VLAN 20, 40, 60, 70, 80 y 100 no se emularon                       |
| Puno (VLAN 130, 10.20.130.0/24)                           | Emulación (junio)          | Implementado en emulación  | Se emuló una sola VLAN de Puno                                                                     |
| Ruteo entre VLAN en el núcleo L3                          | Emulación (junio)          | Implementado               | Núcleo con SVI .1 y ruta por defecto al firewall (ev01)                                            |
| ACL entre zonas (invitados, clínica)                      | Emulación (junio)          | Implementado y probado     | nftables en el núcleo con contadores (ev03, ev04)                                                  |
| Firewall perimetral y DMZ                                 | Emulación (junio)          | Implementado parcialmente  | Firewall Linux con DMZ 10.10.20.0/24 y regla Internet → DMZ solo 443; no es OPNsense ni HA         |
| Firewall perimetral en AWS                                | EC2 inicial en AWS (As-Is) | Implementado (equivalente) | Security Group con 22 (IP del administrador) y 8090; firewall nativo de la nube en lugar de equipo |
| DMZ en la nube (único punto público)                      | EC2 inicial en AWS (As-Is) | Implementado (equivalente) | Nginx publica el ERP y hace proxy de /api; un solo puerto expuesto (8090) en lugar de 443 con TLS  |
| Segmentación en AWS (red Docker, base sin acceso público) | EC2 inicial en AWS (As-Is) | Parcial                    | Red interna de Docker; los puertos internos siguen publicados en el host (riesgo R01 del SGSI)     |
| VPN IPsec Site-to-Site                                    | —                          | Brecha                     | No desplegada; en la emulación el enlace Juliaca–Puno es un enlace ruteado sin cifrado             |
| IDS/IPS (Suricata)                                        | —                          | Brecha                     | No desplegado                                                                                      |
| Honeypot (OpenCanary)                                     | —                          | Brecha                     | No desplegado                                                                                      |
| Monitoreo del ERP (Prometheus, alertas, logs ECS)         | Stack del ERP (CE02 E4)    | Implementado               | 6 reglas de alerta; se ejecuta con el mismo docker-compose que la EC2                              |
| Monitoreo de red (SNMP)                                   | Emulación (junio)          | Implementado en emulación  | snmpwalk al núcleo con SNMPv2c; el diseño exige SNMPv3                                             |

## 3. Implementación

### 3.1 Red de las sedes (emulación)

La topología se construyó con topologia_emulacion.sh: once espacios de nombres de red (Internet, firewall, núcleo, router de Puno, DMZ y equipos de servidor, caja, clínica, invitados y caja de Puno) unidos por enlaces veth, con las subredes del E1. El firewall tiene la salida a Internet (203.0.113.0/30, direccionamiento de documentación), la DMZ 10.10.20.0/24 y el tránsito al núcleo; el núcleo enruta las VLAN 10, 30, 50 y 90 y el enlace hacia Puno (10.20.130.0/24).

| **Equipo emulado** | **Interfaces**                                                           | **Rutas**                                                     |
|--------------------|--------------------------------------------------------------------------|---------------------------------------------------------------|
| fw (firewall)      | 203.0.113.1/30 (Internet) · 172.16.0.1/30 (núcleo) · 10.10.20.1/24 (DMZ) | Por defecto a Internet; 10.10.0.0/16 y 10.20.0.0/16 al núcleo |
| core (núcleo L3)   | 10.10.10.1, 10.10.30.1, 10.10.50.1, 10.10.90.1 (/24) · tránsitos /30     | Por defecto al firewall; 10.20.130.0/24 a Puno                |
| rpuno (Puno)       | 10.20.130.1/24 · 172.16.1.2/30                                           | Por defecto al núcleo                                         |

Control de acceso: nftables en el núcleo (tabla acl) bloquea a invitados hacia servidores, caja, clínica y Puno, y solo deja llegar a la clínica a los servidores y a la propia clínica; en el firewall (tabla perímetro) Internet solo alcanza la DMZ por 443 y nunca la red interna. Ambas reglas aplican las filas Z04, Z10 y Z14 de la matriz del E1.

### 3.2 Perímetro en AWS

| **Recurso**         | **Configuración (según el resumen de la línea)**                                                                    |
|---------------------|---------------------------------------------------------------------------------------------------------------------|
| VPC                 | VPC por defecto 172.31.0.0/16 (el diseño fija VPC-ST 10.50.0.0/16)                                                  |
| EC2                 | Ubuntu 24.04; IP privada 172.31.4.35; Elastic IP pública (documentada como 198.51.100.50\*)                         |
| Security Group      | 22 desde la IP del administrador; 8090 desde 0.0.0.0/0; el resto cerrado                                            |
| Punto de entrada    | Nginx (contenedor frontend) en 8090: sirve la SPA y hace proxy de /api, de modo que el navegador usa un solo origen |
| Acceso a Prometheus | Solo por túnel SSH (puerto 9090 cerrado en el Security Group)                                                       |
| Puertos internos    | 8085, 8087, 9090 y 5433 publicados en el host, protegidos solo por el Security Group (brecha)                       |

## 4. Plan y resultados de pruebas

### 4.1 Pruebas en la emulación

<table>
<colgroup>
<col style="width: 7%" />
<col style="width: 24%" />
<col style="width: 19%" />
<col style="width: 24%" />
<col style="width: 12%" />
<col style="width: 12%" />
</colgroup>
<thead>
<tr class="header">
<th><strong>ID</strong></th>
<th><strong>Prueba</strong></th>
<th><strong>Esperado</strong></th>
<th><strong>Obtenido</strong></th>
<th><strong>Resultado</strong></th>
<th><strong>Regla E1</strong></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>EM-01</td>
<td>Ruteo y tablas de rutas<br />
fw, core, rpuno</td>
<td>Rutas a 10.10.0.0/16 y 10.20.0.0/16 por el núcleo; por defecto al firewall</td>
<td>Tablas como se esperaba</td>
<td>Cumple</td>
<td>§5.2 (ruteo)</td>
</tr>
<tr class="even">
<td>EM-02</td>
<td>Latencia intra-VLAN<br />
Caja 10.10.30.10 → gateway 10.10.30.1</td>
<td>Sin pérdida</td>
<td>10/10 recibidos; 0,091 ms medio</td>
<td>Cumple</td>
<td>OD-06</td>
</tr>
<tr class="odd">
<td>EM-03</td>
<td>Latencia inter-VLAN<br />
Caja (VLAN 30) → servidor (VLAN 10)</td>
<td>Sin pérdida</td>
<td>10/10; 0,079 ms</td>
<td>Cumple</td>
<td>Z05 (equivalente)</td>
</tr>
<tr class="even">
<td>EM-04</td>
<td>Latencia Puno → Juliaca<br />
Caja Puno (VLAN 130) → servidor 10.10.10.10</td>
<td>&lt; 30 ms (OD-06)</td>
<td>10/10; 0,085 ms (el retardo emulado de 18 ms no se aplicó)</td>
<td>No concluyente</td>
<td>OD-06</td>
</tr>
<tr class="odd">
<td>EM-05</td>
<td>Invitados → servidores<br />
10.10.90.10 → 10.10.10.10</td>
<td>Denegado</td>
<td>100 % pérdida; contador de la ACL = 2 paquetes</td>
<td>Cumple</td>
<td>Z14</td>
</tr>
<tr class="even">
<td>EM-06</td>
<td>Caja → servidor<br />
10.10.30.10 → 10.10.10.10</td>
<td>Permitido</td>
<td>2/2 recibidos</td>
<td>Cumple</td>
<td>Z05 (equivalente)</td>
</tr>
<tr class="odd">
<td>EM-07</td>
<td>Caja → clínica<br />
10.10.30.10 → 10.10.50.10</td>
<td>Denegado (aislamiento)</td>
<td>100 % pérdida; contador «bloquea acceso no autorizado a Clínica» = 2</td>
<td>Cumple</td>
<td>Z10</td>
</tr>
<tr class="even">
<td>EM-08</td>
<td>Servidor → clínica<br />
10.10.10.10 → 10.10.50.10</td>
<td>Permitido</td>
<td>2/2 recibidos</td>
<td>Cumple</td>
<td>Matriz §7.3 (servidores)</td>
</tr>
<tr class="odd">
<td>EM-09</td>
<td>Invitados → Internet<br />
10.10.90.10 → 203.0.113.2</td>
<td>Permitido (solo salida)</td>
<td>2/2 recibidos</td>
<td>Cumple</td>
<td>Z15</td>
</tr>
<tr class="even">
<td>EM-10</td>
<td>Rendimiento LAN y Puno (iperf3)<br />
Caja → servidor; Caja Puno → servidor</td>
<td>Puno limitado a 50 Mbit/s por el enlace emulado</td>
<td>14,0 y 13,9 Gbit/s: el límite no se aplicó</td>
<td>No concluyente</td>
<td>§2.1 (ancho de banda)</td>
</tr>
<tr class="odd">
<td>EM-11</td>
<td>Monitoreo SNMP del núcleo<br />
NOC → core-sw-juliaca</td>
<td>MIB-II e IF-MIB legibles</td>
<td>sysName, interfaces y contadores leídos</td>
<td>Cumple (con observación: SNMPv2c)</td>
<td>§7 / Z12</td>
</tr>
</tbody>
</table>

### 4.2 Pruebas del perímetro en AWS

| **ID** | **Prueba**                         | **Esperado**                 | **Estado**                                                                                  |
|--------|------------------------------------|------------------------------|---------------------------------------------------------------------------------------------|
| P01    | Internet → 8090                    | Permitido, HTTP 200          | Procedimiento de validación documentado (§7)                                                |
| P02    | Internet → 8085                    | Denegado                     | Procedimiento documentado (nmap externo, §7)                                                |
| P03    | Internet → 5433                    | Denegado                     | Procedimiento documentado (nmap externo, §7)                                                |
| P04    | Internet → 9090                    | Denegado                     | Procedimiento documentado (nmap externo, §7)                                                |
| P05    | SSH desde IP no autorizada         | Denegado                     | Pendiente                                                                                   |
| P06    | SSH desde la IP del administrador  | Permitido                    | Usado en la operación (incidentes registrados en el resumen); sin captura                   |
| P07    | Frontend → backend por red interna | {"status":"UP"}              | Verificado con el mismo docker-compose en local: health UP en las pruebas de humo (CE02 E4) |
| P08    | Login con credenciales inválidas   | 401 y bloqueo tras el límite | Verificado en local: 401 (Postman 02, CE02 E4)                                              |
| P09    | Suite Postman contra la IP pública | Aserciones aprobadas         | Pendiente; en local 19/19 peticiones y 41/41 aserciones                                     |
| P10    | Prometheus por túnel SSH           | Panel visible                | Pendiente de captura; en local, 6 reglas cargadas (CE02 E4 figura 2.1)                      |

### 4.3 Matriz de pruebas del diseño (C01–C20)

Las 20 pruebas de conectividad del E1 v2.1 están en la hoja Matriz_C01_C20 del XLSX con su columna de estado. Todas siguen pendientes: necesitan la VPN, las VLAN de las dos sedes y los firewalls OPNsense, que no se desplegaron. Las pruebas de la emulación cubren en forma equivalente las reglas Z04, Z05, Z10, Z14 y Z15.

### 4.4 Validez de la evidencia

| **Evidencia**                                      | **Validez**          | **Motivo**                                                                                                                              |
|----------------------------------------------------|----------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| ev01–ev06 (configuración, ping, ACL, iperf3, SNMP) | Válida               | Salidas de herramientas estándar sobre la emulación con network namespaces                                                              |
| ev02 y ev05 en el tramo Juliaca–Puno               | Válida con reserva   | El retardo y el límite de ancho de banda no se aplicaron: no representan un enlace WAN                                                  |
| ev07 (trazas preliminares)                         | Excluida             | Trazas genéricas preliminares no aplicadas a la topología Linux; se conserva evidencia verificable de herramientas estándar (ev01–ev06) |
| configuraciones/\*.cfg (Cisco IOS/ASA)             | Diseño, no ejecución | Configuraciones de referencia; no se cargaron en equipos reales                                                                         |
| Datos de la EC2 (resumen de la línea)              | Declarativa          | Despliegue operativo en AWS; procedimiento de validación documentado en §7                                                              |

## 5. Monitoreo y visibilidad

- **SNMP:** snmpwalk desde el NOC al núcleo leyó sysName, la ubicación, las interfaces y los contadores de octetos (EM-11). Se usó SNMPv2c con comunidad de solo lectura; el diseño exige SNMPv3 con autenticación y cifrado.

- **ERP:** Prometheus recoge las métricas del backend y de negocio y evalúa 6 reglas de alerta (caída, 5xx, latencia de ventas, eventos outbox pendientes o con error, órdenes de reabastecimiento); Actuator expone la salud y Docker aplica healthchecks. Verificado con el mismo stack en local (CE02 E4); en la EC2 se consulta por túnel SSH.

- **Registros:** el backend escribe logs JSON en formato ECS que se leen con docker compose logs; la centralización en un SIEM (Wazuh, E1 §8) no se desplegó. El syslog de la emulación (ev07) se excluyó como evidencia (sección 4.4).

## 6. Incidencias detectadas

| **Incidencia**                                                       | **Clasif.** | **Contención y solución**                                                                | **Lección**                                     |
|----------------------------------------------------------------------|-------------|------------------------------------------------------------------------------------------|-------------------------------------------------|
| Error 502 en el login tras recrear el backend                        | Media       | Esperar el estado healthy y reiniciar el frontend para que Nginx resuelva la nueva IP    | Usar resolver 127.0.0.11 en Nginx               |
| Falla al cargar la llave JWT                                         | Alta        | Generar el par RSA en secrets/jwt y recrear los servicios                                | Documentar los secretos que no viajan por git   |
| Carpeta secrets creada por Docker como root                          | Baja        | chown al usuario ubuntu                                                                  | Crear las carpetas antes de levantar el compose |
| CORS 403 detrás de Nginx (en local)                                  | Media       | proxy_set_header Host \$http_host y orígenes por variable CORS_ALLOWED_ORIGINS           | No fijar orígenes en el código                  |
| Emulación: tc/netem no aplicó retardo ni límite en el enlace de Puno | Baja        | Los errores se ocultaban con «\|\| true»; las mediciones WAN se declaran no concluyentes | No silenciar errores en scripts de prueba       |

## 7. Procedimiento técnico de ejecución y validación de pruebas

Procedimiento técnico de ejecución y comandos de validación operativa (P01–P10) en AWS:

nmap -Pn -p 22,8085,8087,8090,9090,5433 \<elastic-ip\>

curl -I http://\<elastic-ip\>:8090

ssh -i llave.pem ubuntu@\<elastic-ip\> 'docker compose ps; ss -tlnp'

docker compose exec frontend wget -qO- http://backend:8085/actuator/health

postman collection run pruebas/postman/E3_SaludTotal.postman_collection.json --env-var base_url=http://\<elastic-ip\>:8090

ssh -i llave.pem -L 9090:localhost:9090 ubuntu@\<elastic-ip\> (abrir http://localhost:9090/alerts)

## 8. Brechas y plan de cierre

| **Brecha**                          | **Acción mínima viable**                                                                                    | **Esfuerzo** |
|-------------------------------------|-------------------------------------------------------------------------------------------------------------|--------------|
| VPN IPsec Site-to-Site              | Túnel entre la EC2 y una VM local (strongSwan o WireGuard como equivalencia técnica) o AWS Site-to-Site VPN | 3 a 4 h      |
| IDS/IPS                             | Suricata o fail2ban en la EC2 y alertas de un escaneo nmap autorizado                                       | 1 a 2 h      |
| Honeypot                            | OpenCanary o Cowrie en un contenedor aislado                                                                | 1 h          |
| Puertos publicados en el host       | Publicar 8085, 8087, 9090 y 5433 en 127.0.0.1 en el compose                                                 | 15 min       |
| HTTPS                               | Caddy delante de Nginx con un dominio gratuito (sslip.io o DuckDNS)                                         | 1 h          |
| Pruebas P01–P10                     | Ejecutar los comandos de la sección 7 y guardar las salidas                                                 | 30 min       |
| SNMPv3 y registro centralizado real | rsyslog o Wazuh recibiendo eventos reales del firewall y del núcleo                                         | 2 h          |
| Emulación WAN fiel                  | Repetir EM-04 y EM-10 con tc/netem aplicado y verificado                                                    | 30 min       |

## 9. Conclusiones

La segmentación y el control de acceso del diseño funcionan: la emulación demuestra con contadores que invitados y caja no alcanzan servidores ni clínica, y que Internet solo llega a la DMZ por 443. En la nube, el ERP se publica por un único puerto con su backend detrás de Nginx. Lo que el diseño pide y no está implementado —VPN, IDS/IPS, honeypot, TLS, puertos internos cerrados en el host y SNMPv3— se declara como brecha con su acción de cierre, no como logrado. Las mediciones del enlace WAN emulado no son válidas y deben repetirse.

## 10. Referencias

\[1\] IEEE Std 802.1Q-2022, "Bridges and Bridged Networks", IEEE, 2022.

\[2\] NIST SP 800-41 Rev. 1, "Guidelines on Firewalls and Firewall Policy", NIST, 2009.

\[3\] NIST SP 800-115, "Technical Guide to Information Security Testing and Assessment", NIST, 2008.

\[4\] The netfilter project, "nftables wiki", 2026. \[En línea\]. Disponible: https://wiki.nftables.org/

\[5\] iproute2 / tc-netem(8), Linux manual pages, 2026.

\[6\] ESnet, "iperf3 documentation", 2026.

\[7\] J. Case et al., "A Simple Network Management Protocol (SNMP)", RFC 1157; U. Blumenthal y B. Wijnen, "User-based Security Model (USM) for SNMPv3", RFC 3414.

\[8\] Amazon Web Services, "Amazon VPC User Guide — Security groups", 2026.

## Anexo A. Índice de evidencias

| **Archivo**                                                                | **Contenido**                                             |
|----------------------------------------------------------------------------|-----------------------------------------------------------|
| version_anterior_v1/Evidencias_Red_SaludTotal.zip · topologia_emulacion.sh | Construcción de la topología emulada                      |
| · pruebas/ev01_config_routing.txt                                          | Interfaces y tablas de rutas (EM-01)                      |
| · pruebas/ev02_conectividad_latencia.txt                                   | ping y traceroute (EM-02 a EM-04)                         |
| · pruebas/ev03_acl_config.txt                                              | Reglas nftables del núcleo y del firewall                 |
| · pruebas/ev04_pruebas_acl.txt                                             | Pruebas de ACL con contadores (EM-05 a EM-09)             |
| · pruebas/ev05_throughput.txt                                              | iperf3 (EM-10, no concluyente)                            |
| · pruebas/ev06_snmp.txt                                                    | snmpwalk al núcleo (EM-11)                                |
| · pruebas/ev07_logs.txt                                                    | Excluida (sección 4.4)                                    |
| Resumen de la línea CE03, §5                                               | Perímetro y servicio en AWS                               |
| CE02 E4 (articulos-e3/ci/reportes, pruebas/postman)                        | Health UP, 401 y alertas de Prometheus con el mismo stack |


---

### Archivos del entregable

- [:material-file-word: Descargar documento original (.docx)](Entregable_S2E2_Red_SaludTotal.docx)

**Anexos**

- [:material-file-excel: Matriz_Pruebas_Red_S2E2.xlsx](archivos/Anexos/Matriz_Pruebas_Red_S2E2.xlsx)
