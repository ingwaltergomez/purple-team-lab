# Troubleshooting — Wazuh Purple Team Lab

**Fecha:** 2026-09-22
**Autor:** Walter Gómez
**Proyecto:** Purple Team Lab — Wazuh + Sysmon + Windows 11
**Objetivo:** Documentar la resolución de tres fallos en cascada que impidieron el funcionamiento del laboratorio durante la instalación inicial de Wazuh.

---

## Resumen Ejecutivo

Durante la instalación del laboratorio de Purple Team, el Wazuh Manager no podía sincronizar con el Indexer, el Dashboard rechazaba el login, y no se generaban alertas. Los tres síntomas parecían independientes, pero tenían una causa raíz común: en algún momento se regeneraron los certificados y la CA de Wazuh, pero esa nueva generación nunca se propagó completamente a todos los componentes. Esto dejó al Manager, al Dashboard y a Filebeat usando certificados y credenciales de generaciones distintas. La resolución requirió alinear los cuatro componentes (Indexer, Manager, Dashboard, Filebeat) a la misma generación de certificados y credenciales.

---

## Arquitectura del laboratorio

| Componente | Rol | IP |
|------------|-----|-----|
| Proxmox VE 9.2 | Hipervisor (miniPC bare-metal) | 192.168.1.100 |
| Wazuh Indexer + Manager + Dashboard | SIEM (VM Ubuntu 24.04) | 192.168.1.101 |
| Windows 11 | Endpoint víctima (con Sysmon y agente Wazuh) | 192.168.1.5 |

---

## Problema 1: IndexerConnector initialization failed

### Síntoma

El log del Manager mostraba repetidamente:

    indexer-connector: WARNING: IndexerConnector initialization failed for index 'wazuh-states-vulnerabilities-wazuh-lab', retrying until the connection is successful.

El curl con usuario admin y contraseña nueva sí funcionaba contra el Indexer. Pero el Manager no podía conectarse.

### Diagnóstico

El certificado admin.pem que usaba el Manager estaba firmado por una CA antigua. El root-ca.pem en /etc/filebeat/certs/ era de una generación distinta a la que usaba el Indexer. El wazuh-keystore del Manager no tenía configurada la contraseña.

### Causa raíz

Cuando se regeneraron los certificados con wazuh-certs-tool.sh, los nuevos certificados se aplicaron al Indexer, pero el Manager siguió usando los certificados viejos que estaban en /etc/filebeat/certs/. Como el Indexer ya no reconocía la CA antigua, rechazaba la conexión del Manager.

### Solución

Paso 1: Copiar el root-ca.pem correcto desde el Indexer al Manager:

    cp /etc/wazuh-indexer/certs/root-ca.pem /etc/filebeat/certs/root-ca.pem

Paso 2: Copiar el admin.pem y admin-key.pem correctos desde el directorio de certificados generados:

    cp /root/wazuh-certificates/admin.pem /etc/filebeat/certs/admin.pem
    cp /root/wazuh-certificates/admin-key.pem /etc/filebeat/certs/admin-key.pem
    chmod 400 /etc/filebeat/certs/admin*.pem

Paso 3: Configurar el keystore del Manager con el usuario y contraseña:

    echo 'admin' | /var/ossec/bin/wazuh-keystore -f indexer -k username
    echo 'TU_PASSWORD' | /var/ossec/bin/wazuh-keystore -f indexer -k password

Paso 4: Reiniciar el Manager:

    systemctl restart wazuh-manager

### Verificación

    grep "indexer-connector" /var/ossec/logs/ossec.log | tail -3

Resultado esperado:

    indexer-connector: INFO: IndexerConnector initialized successfully for index: wazuh-states-vulnerabilities-wazuh-lab.

---

## Problema 2: Login del Dashboard fallaba

### Síntoma

El login en https://192.168.1.101 con el usuario admin y la contraseña establecida devolvía error de autenticación.

### Diagnóstico

El certificado admin.pem autentica por client-cert, no por contraseña. Por eso el Manager podía funcionar sin que la contraseña del Dashboard fuera correcta. El hash de la contraseña de admin en internal_users.yml nunca se actualizó después del cambio de contraseña.

### Causa raíz

El archivo internal_users.yml del Indexer seguía teniendo el hash de la contraseña original. El cambio manual de contraseña no se había aplicado al Indexer.

### Solución

Paso 1: Generar el hash de la nueva contraseña:

    export JAVA_HOME=/usr/share/wazuh-indexer/jdk
    bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh -p 'NuevaContraseña'

Paso 2: Actualizar internal_users.yml con el nuevo hash para los usuarios admin y wazuh-server:

    nano /etc/wazuh-indexer/opensearch-security/internal_users.yml

Paso 3: Aplicar los cambios con securityadmin.sh:

    sudo JAVA_HOME=/usr/share/wazuh-indexer/jdk bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd /etc/wazuh-indexer/opensearch-security/ -icl -nhnv -h 192.168.1.101 -cacert /etc/wazuh-indexer/certs/root-ca.pem -cert /etc/wazuh-indexer/certs/admin.pem -key /etc/wazuh-indexer/certs/admin-key.pem

### Verificación

Login exitoso en el Dashboard con la nueva contraseña.

---

## Problema 3: Índice de alertas faltante (Filebeat caído)

### Síntoma

El Dashboard mostraba "No results match your search criteria" porque el índice wazuh-alerts-* no existía.

### Diagnóstico

El servicio filebeat estaba caído. Filebeat buscaba wazuh-1.pem en /etc/filebeat/certs/, pero ese archivo no existía. Filebeat usaba su propio keystore (${username}/${password} en filebeat.yml), independiente del wazuh-keystore del Manager. El keystore de Filebeat tenía credenciales viejas.

### Causa raíz

El certificado wazuh-1.pem se había movido a /etc/filebeat/certs.bak/ durante una migración de certificados anterior. Filebeat no lo encontraba, no podía autenticarse, y no enviaba eventos al Indexer.

### Solución

Paso 1: Localizar el certificado en el backup:

    ls -la /etc/filebeat/certs.bak/

Paso 2: Validar el certificado contra la CA correcta:

    openssl verify -CAfile /etc/wazuh-indexer/certs/root-ca.pem /etc/filebeat/certs.bak/wazuh-1.pem

Resultado esperado: OK.

Paso 3: Copiar el certificado y la clave al directorio activo:

    cp /etc/filebeat/certs.bak/wazuh-1.pem /etc/filebeat/certs/
    cp /etc/filebeat/certs.bak/wazuh-1-key.pem /etc/filebeat/certs/
    chmod 400 /etc/filebeat/certs/wazuh-1*.pem

Paso 4: Actualizar el keystore de Filebeat con las credenciales correctas:

    filebeat keystore create
    echo 'admin' | filebeat keystore add username --stdin --force
    echo 'TU_PASSWORD' | filebeat keystore add password --stdin --force

Paso 5: Reiniciar Filebeat:

    systemctl restart filebeat

### Verificación

    filebeat test output

Resultado esperado:

    talk to server... OK

Y verificar que el índice wazuh-alerts-* existe:

    curl -k -u admin:TU_PASSWORD "https://192.168.1.101:9200/_cat/indices/wazuh-alerts-*?v"

---

## Lecciones Aprendidas

### 1. Regenerar certificados requiere redistribución completa

Cuando se regeneran los certificados con wazuh-certs-tool.sh, hay que redistribuir el set completo a todos los componentes simultáneamente. El Indexer usa node-1.pem, node-1-key.pem y root-ca.pem en /etc/wazuh-indexer/certs/. El Manager usa admin.pem, admin-key.pem y root-ca.pem en /etc/filebeat/certs/. El Dashboard usa dashboard.pem, dashboard-key.pem y root-ca.pem en /etc/wazuh-dashboard/certs/. Filebeat usa wazuh-1.pem, wazuh-1-key.pem y root-ca.pem en /etc/filebeat/certs/. Si un solo componente queda con certificados de una generación anterior, el laboratorio falla en cascada.

### 2. Los keystores son independientes

Wazuh tiene dos keystores distintos. El /var/ossec/bin/wazuh-keystore es para el Manager (conexión al Indexer). El filebeat keystore es para Filebeat (envío de eventos al Indexer). Actualizar uno no actualiza el otro. Hay que configurar ambos.

### 3. La autenticación por certificado es silenciosa

El Manager puede autenticarse por certificado sin usar contraseña. Eso hace que los fallos de contraseña no se manifiesten hasta que intentas acceder al Dashboard.

### 4. Verificar la CA antes de culpar a los componentes

Antes de asumir que un componente está roto, verifica que los certificados estén firmados por la CA correcta:

    openssl verify -CAfile /etc/wazuh-indexer/certs/root-ca.pem /etc/filebeat/certs/admin.pem

Si devuelve OK, el certificado es válido. Si devuelve error, el certificado es de otra generación.

---

## Recomendaciones para el futuro

1. No regenerar certificados sin un plan de redistribución. Si lo haces, hazlo en todos los componentes a la vez.

2. Limpiar los directorios de backups después de una migración exitosa:

    mv /etc/filebeat/certs.bak /etc/filebeat/certs.bak.obsoleto-$(date +%Y%m%d)
    mv /root/wazuh-certificates /root/wazuh-certificates.obsoleto-$(date +%Y%m%d)

3. Documentar cada generación de certificados con un timestamp y un fingerprint de la CA:

    openssl x509 -in /etc/wazuh-indexer/certs/root-ca.pem -noout -fingerprint -sha256

Guardar ese fingerprint en un archivo de referencia. Si en el futuro algo falla, comparar el fingerprint actual con el guardado.

4. Verificar los cuatro componentes después de cualquier cambio de certificados. Verificar el Indexer con systemctl status wazuh-indexer. Verificar el Manager con grep "indexer-connector" /var/ossec/logs/ossec.log | tail -3. Verificar el Dashboard con curl -k -u admin:TU_PASSWORD https://192.168.1.101:9200/_cluster/health. Verificar Filebeat con filebeat test output.

---

## Conclusión

Los tres fallos del laboratorio tenían una causa común: inconsistencia de certificados entre generaciones. La resolución requirió alinear los cuatro componentes (Indexer, Manager, Dashboard, Filebeat) a la misma generación de certificados y credenciales. La lección más importante es que Wazuh no propaga automáticamente los cambios de certificados entre componentes. Cada componente tiene su propio conjunto de certificados y su propio keystore. Si uno se actualiza sin los demás, el laboratorio falla. Este documento sirve como referencia para futuras instalaciones y como evidencia del proceso de troubleshooting para el portafolio de Purple Team.

---

*Documento preparado como parte del Purple Team Lab del portafolio técnico de Walter Gómez.*
