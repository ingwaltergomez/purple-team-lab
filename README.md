# Purple Team Lab — Wazuh + Sysmon

Laboratorio de validación de detección con técnicas reales de MITRE ATT&CK en un endpoint Windows.

**Autor:** Walter Gómez
**Fecha:** Septiembre 2026
**Artículo en Medium:** [Construí un Purple Team Lab con Wazuh + Sysmon](https://medium.com/@waltgomez/constru%C3%AD-un-purple-team-lab-con-wazuh-sysmon-esto-es-lo-que-detect%C3%B3-604650ceeba2)

---

## Resumen Ejecutivo

Se construyó un laboratorio de Purple Team con Proxmox VE, una VM Windows 11 como endpoint víctima, Sysmon como fuente de telemetría, y Wazuh como SIEM. Se ejecutaron 4 técnicas de MITRE ATT&CK usando Atomic Red Team, y se midió la capacidad de detección de Wazuh.

**Resultados clave:**

- **T1059.001 (PowerShell)** — Detectada. Reglas 92004, 92032, 92052.
- **T1082 (System Information Discovery)** — Pendiente de verificación.
- **T1057 (Process Discovery)** — Pendiente de verificación.
- **Tampering de Defender (T1562.001)** — Detectada. Reglas de nivel 12 y 13.

La comparación entre el escenario sin SIEM (Base 1) y el escenario con SIEM (Base 3) demuestra que Wazuh no solo detecta las técnicas, sino que las clasifica con mapeo MITRE ATT&CK y proporciona telemetría forense completa.

---

## Arquitectura del Laboratorio

| Componente | Rol | IP |
|------------|-----|-----|
| Proxmox VE 9.2 | Hipervisor (miniPC bare-metal) | 192.168.1.100 |
| Wazuh Indexer + Manager + Dashboard | SIEM (VM Ubuntu 24.04) | 192.168.1.101 |
| Windows 11 | Endpoint víctima (con Sysmon y agente Wazuh) | 192.168.1.5 |

### Stack de Telemetría

- **Sysmon:** Captura eventos de creación de procesos, conexiones de red, y cambios en archivos.
- **Agente Wazuh:** Lee el canal `Microsoft-Windows-Sysmon/Operational` y envía los eventos al Manager.
- **Wazuh Manager:** Procesa los eventos, aplica reglas de detección, y los envía al Indexer.
- **Wazuh Indexer:** Almacena los eventos y los hace consultables desde el Dashboard.
- **Wazuh Dashboard:** Interfaz de visualización y búsqueda (Threat Hunting).

---

## Estructura del Repositorio

purple-team-lab/
├── README.md
├── informe-purple-team.md
├── informe-purple-team.pdf
├── informe-purple-team.tex
├── troubleshooting-wazuh.md
└── capturas/
    ├── diagrama-arquitectura.png
    ├── tabla-comparativa-b1-vs-b3.png
    ├── tabla-reglas.png
    ├── tabla-tecnicas-ejecutadas.png
    ├── fase01-atomic-red-team-instalado.png
    ├── fase01-windows-baseline-sin-defensa.png
    ├── fase01-windows-baseline-sin-defensa_02.png
    ├── fase01-windows-bypassnro.png
    ├── fase01-windows-bypassnro-no-internet.png
    ├── fase01-windows-desktop.png
    ├── fase01-windows-exclusiones.png
    ├── fase01-windows-snapshot-limpio.png
    ├── fase01-windows-summary-ip.png
    ├── fase01-windows-virtio-scsi.png
    ├── fase01-windows-virtio-tools.png
    ├── fase02-defender-deteccion-amsi.png
    ├── fase02-defender-exclusiones-ampliadas.png
    ├── fase03-filebeat-connector.png
    ├── fase03-indexer-connector-ok.png
    ├── fase03-indice-alertas-creado.png
    ├── fase03-sysmon-detecta-ataque.png
    ├── fase03-sysmon-eventos.png
    ├── fase03-sysmon-instalado.png
    ├── fase03-wazuh-agente-activo.png
    ├── fase03-wazuh-agente-leyendo-sysmon.png
    ├── fase03-wazuh-agente-windows-running.png
    ├── fase03-wazuh-dashboard-instalado.png
    ├── fase03-wazuh-detalle-evento.png
    ├── fase03-wazuh-detalle-evento-completo.png
    ├── fase03-wazuh-detalle-regla-mitre.png
    ├── fase03-wazuh-detecta-ataque.png
    ├── fase03-wazuh-eventos-reales.png
    ├── fase03-wazuh-hash-password.png
    ├── fase03-wazuh-password-cambiada.png
    ├── fase03-wazuh-sysmon-config.png
    ├── fase03-wazuh-sysmon-leyendo.png
    ├── fase03-wazuh-t1057.png
    ├── fase03-wazuh-t1059-001-10.png
    ├── fase03-wazuh-t1082.png
    └── fase03-wazuh-vm-instalada.png

---

## Artefactos

### 1. Informe de Purple Team

`informe-purple-team.md` — Informe completo con metodología, técnicas ejecutadas, detecciones observadas, análisis de gaps, y comparación Base 1 vs Base 3.

También disponible en PDF (`informe-purple-team.pdf`) y LaTeX (`informe-purple-team.tex`).

### 2. Documento de Troubleshooting

`troubleshooting-wazuh.md` — Resolución de tres fallos en cascada que impidieron el funcionamiento del laboratorio durante la instalación inicial de Wazuh:

1. **IndexerConnector initialization failed** — Causa raíz: inconsistencia de certificados entre generaciones.
2. **Login del Dashboard fallaba** — Causa raíz: hash de contraseña no actualizado en `internal_users.yml`.
3. **Índice de alertas faltante** — Causa raíz: Filebeat caído por certificado `wazuh-1.pem` faltante.

### 3. Capturas del Laboratorio

11 capturas que documentan el proceso completo: instalación, configuración, ejecución de técnicas, y detecciones.

---

## Técnicas Ejecutadas y Detecciones Observadas

| Técnica | MITRE ID | Comando | Regla Wazuh | Detectado? |
|---------|----------|---------|-------------|------------|
| PowerShell | T1059.001 | `Invoke-AtomicTest T1059.001 -TestNumbers 8` | 92004, 92032, 92052 | ✅ |
| Fileless Script Execution | T1059.001 | `Invoke-AtomicTest T1059.001 -TestNumbers 10` | 92004 | ✅ |
| System Information Discovery | T1082 | `Invoke-AtomicTest T1082 -TestNumbers 1` | 92053 | Pendiente |
| Process Discovery | T1057 | `Invoke-AtomicTest T1057 -TestNumbers 1` | 92031 | Pendiente |
| Defender Tampering | T1562.001 | Defender Control | 12, 13 | ✅ |

### Detalle de las Reglas que Dispararon

| Regla | Descripción | Nivel | MITRE |
|-------|-------------|-------|-------|
| 92004 | Powershell process spawned Windows command shell instance | 4 | ✅ |
| 92032 | Suspicious Windows cmd shell execution | 3 | ✅ |
| 92052 | Windows command prompt started by an abnormal process | 4 | ✅ |
| 12 | Windows Defender real time monitoring was disabled by Powershell command | 12 | ✅ |
| 13 | Windows Defender Intrusion prevention system was disabled by Powershell command | 13 | ✅ |

---

## Comparación Base 1 vs Base 3

| Capacidad | Base 1 (sin SIEM) | Base 3 (con SIEM) |
|-----------|-------------------|-------------------|
| Detección | ✅ (Defender) | ✅ (Wazuh) |
| Telemetría forense | ❌ | ✅ |
| Mapeo MITRE | ❌ | ✅ |
| Consultas históricas | ❌ | ✅ |
| Dashboard | ❌ | ✅ |
| Correlación | ❌ | ✅ |

**Conclusión:** El SIEM no reemplaza al antivirus, lo complementa. Defender bloquea, pero no te dice qué pasó. Wazuh te dice exactamente qué pasó, cuándo, cómo, y con qué técnica de MITRE ATT&CK.

---

## Herramientas Utilizadas

| Herramienta | Propósito |
|-------------|-----------|
| **Proxmox VE 9.2** | Hipervisor |
| **Wazuh 4.12** | SIEM (Indexer + Manager + Dashboard) |
| **Sysmon** | Telemetría de endpoint Windows |
| **Atomic Red Team** | Ejecución de técnicas MITRE ATT&CK |
| **Windows 11** | Endpoint víctima |
| **Ubuntu Server 24.04** | VM del SIEM |

---

## Lecciones Aprendidas

1. **Wazuh detecta técnicas reales de MITRE ATT&CK** con reglas específicas y mapeo MITRE.
2. **Sysmon es la fuente de telemetría crítica.** Sin Sysmon, Wazuh no tendría visibilidad de procesos.
3. **La combinación Sysmon + Wazuh proporciona telemetría forense completa**, superando la capacidad de un antivirus tradicional.
4. **La detección de tampering de Defender** (reglas 12 y 13) demuestra que Wazuh tiene reglas para detectar la desactivación de controles de seguridad.
5. **La inconsistencia de certificados entre generaciones** es una causa raíz común de fallos en cascada en Wazuh.

---

## Autor

**Walter Gómez**
Especialista en Infraestructura y Seguridad
ISO 27001 Lead Implementer | DevSecOps Engineer
Más de 20 años de experiencia en Linux y Red Hat

- LinkedIn: [https://www.linkedin.com/in/ingwaltergomez](https://www.linkedin.com/in/ingwaltergomez)
- Medium: [https://medium.com/@waltgomez](https://medium.com/@waltgomez)
- GitHub: [https://github.com/ingwaltergomez](https://github.com/ingwaltergomez)

---

## Licencia

Este proyecto está bajo la licencia MIT.

---

*Documento preparado como parte del portafolio técnico de Walter Gómez. Proyecto 2: Purple Team Lab.*
