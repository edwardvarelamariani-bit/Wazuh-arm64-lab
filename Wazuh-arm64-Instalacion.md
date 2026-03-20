# Wazuh 4.14 en ARM64 — Guía de Instalación Nativa

> **Plataforma:** Apple Silicon (M1/M2/M3) con UTM · Ubuntu 25.10 ARM64  
> **Versión Wazuh:** 4.14.4  
> **Método:** Instalación nativa sin Docker (recomendado para ARM64)  
> **Autor:** Edward  

---

## ¿Por qué instalación nativa y no Docker?

Wazuh en Docker sobre ARM64 presenta problemas importantes:

- El **Wazuh Dashboard** no tiene imagen Docker oficial para ARM64
- El **Wazuh Indexer** (OpenSearch) en Docker sobre ARM64 requiere emulación x86_64, lo que lo hace extremadamente lento
- La instalación nativa con paquetes `.deb` soporta **ARM64 de forma oficial** desde Wazuh 4.x

---

## Notas importantes antes de empezar

### Copiar y pegar en la terminal de Ubuntu
En la terminal de Linux **no funciona** `Ctrl+C` / `Ctrl+V` para copiar/pegar. Usa:
- **Copiar:** `Ctrl+Shift+C`
- **Pegar:** `Ctrl+Shift+V`

### Sobre el disco externo USB
Si tienes la VM en un disco externo USB (HDD o SSD), ten en cuenta que:
- Los tiempos de instalación se **multiplican significativamente**
- Durante la instalación del Manager (el paso más pesado), Ubuntu puede parecer completamente bloqueado durante 20-30 minutos
- **No canceles el proceso** — el disco parpadeando es señal de que sigue trabajando
- Se recomienda disco interno o SSD externo para mejores tiempos

### Versión del script de instalación
Asegúrate siempre de descargar la versión **4.14** o superior del script. Las versiones anteriores (4.11 y anteriores) no tienen soporte ARM64 completo y pueden fallar silenciosamente descargando paquetes x86_64.

---

## Requisitos previos

| Componente | Mínimo recomendado |
|------------|-------------------|
| RAM | 6 GB |
| Disco | 50 GB libres |
| CPU | 4 núcleos |
| SO | Ubuntu 20.04 / 22.04 / 25.10 (ARM64) |
| Arquitectura | aarch64 (ARM64) |

Verifica tu arquitectura antes de empezar:

```bash
uname -m
# Debe mostrar: aarch64
```

---

## Configuración de red en UTM (casa WiFi vs clase cable)

Si usas UTM en Mac con diferentes redes según el entorno, configura dos interfaces de red para no tener que cambiar nada manualmente.

### Añadir segunda interfaz NAT en UTM

1. Apaga la VM
2. UTM → Edit → **+ New Device** → **Network** → **Shared Network (NAT)**
3. Arranca la VM

### Configurar Netplan

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  renderer: NetworkManager
  ethernets:
    enp0s1:          # Bridge - cable ethernet
      dhcp4: true
      dhcp6: true
      optional: true
      match:
        macaddress: XX:XX:XX:XX:XX:XX    # MAC de tu interfaz Bridge
      set-name: enp0s1
    enp0s2:          # NAT - WiFi
      dhcp4: true
      dhcp6: true
      optional: true
      match:
        macaddress: XX:XX:XX:XX:XX:XX    # MAC de tu interfaz NAT
      set-name: enp0s2
  version: 2
```

> ⚠️ Sustituye las MACs por las tuyas. Puedes verlas con `ip a`.

```bash
sudo netplan generate
sudo netplan apply
```

### Evitar el error "Fallo la conexión" al arrancar

```bash
sudo nmcli connection modify netplan-enp0s1 connection.autoconnect no
sudo nmcli connection modify netplan-enp0s1 ipv4.may-fail yes
sudo nmcli connection modify netplan-enp0s1 ipv6.may-fail yes
sudo nmcli connection delete "Conexión cableada 1"   # Si existe
```

Con esto:
- **En casa (WiFi):** La VM arranca sin errores, sale por NAT (enp0s2)
- **En clase (cable):** La VM coge IP por Bridge (enp0s1) automáticamente

---

## Ampliación del disco en UTM (recomendado)

Wazuh necesita aproximadamente 10-15GB. Si tu VM tiene poco espacio libre, amplía el disco antes de instalar.

### En UTM (con la VM apagada)
1. Apaga la VM: `sudo shutdown now`
2. UTM → Edit → selecciona el disco (**VirtIO Drive**)
3. Cambia el tamaño (recomendado mínimo **150GB**)
4. Guarda y arranca la VM

### Dentro de Ubuntu — expandir la partición
```bash
# Verificar el disco
lsblk

# Expandir la partición (normalmente vda2)
sudo growpart /dev/vda 2

# Expandir el sistema de ficheros
sudo resize2fs /dev/vda2

# Verificar
df -h /
```

---

## Instalación de Wazuh

### Paso 1 — Descargar el script de instalación

```bash
cd ~
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
```

> ⚠️ Asegúrate de descargar la versión **4.14** o superior. Versiones anteriores no tienen soporte ARM64 completo.

### Paso 2 — Configurar las IPs

Edita el `config.yml` con tu IP (sustitúyela por la tuya):

```bash
sed -i 's/<indexer-node-ip>/192.168.64.6/g; s/<wazuh-manager-ip>/192.168.64.6/g; s/<dashboard-node-ip>/192.168.64.6/g' config.yml
```

Verifica que quedó bien:

```bash
grep "ip:" config.yml
```

Debe mostrar tu IP en los tres campos.

### Paso 3 — Generar certificados

```bash
sudo bash wazuh-install.sh --generate-config-files
```

> ℹ️ Ignorar el WARNING sobre sistemas recomendados — Ubuntu 25.10 funciona perfectamente aunque no esté en la lista oficial.

Debe terminar con:
```
INFO: Created wazuh-install-files.tar.
```

### Paso 4 — Instalar el Wazuh Indexer

```bash
sudo bash wazuh-install.sh --wazuh-indexer node-1
```

⏱️ Tarda aproximadamente **5-10 minutos**.

Debe terminar con:
```
INFO: Installation finished.
```

#### ⚠️ Problema habitual en ARM64: Timeout del Indexer

Si al arrancar el Indexer obtienes un error de timeout:

```bash
sudo mkdir -p /var/log/wazuh-indexer
sudo chown -R wazuh-indexer:wazuh-indexer /var/log/wazuh-indexer
sudo chmod 750 /var/log/wazuh-indexer
```

Ampliar el timeout de systemd a 600 segundos:

```bash
sudo systemctl edit wazuh-indexer
```

Añade esto en el editor:

```ini
[Service]
TimeoutStartSec=600
```

Guarda con `Ctrl+O` → `Enter` → `Ctrl+X`, luego:

```bash
sudo systemctl daemon-reload
sudo systemctl start wazuh-indexer
```

### Paso 5 — Instalar el Wazuh Manager

```bash
sudo bash wazuh-install.sh --wazuh-server wazuh-1
```

⏱️ Este es el paso más largo — puede tardar **20-30 minutos** en ARM64.

> ℹ️ Durante la instalación Ubuntu puede parecer bloqueado. Es normal — el Manager consume todos los recursos disponibles. No canceles el proceso.

Debe terminar con:
```
INFO: Installation finished.
```

### Paso 6 — Instalar el Wazuh Dashboard

```bash
sudo bash wazuh-install.sh --wazuh-dashboard dashboard -fd
```

> ℹ️ El flag `-fd` (force dashboard) evita que el script espere al Indexer y se autodesinstale si tarda en conectar.

#### ⚠️ Problema habitual: directorio de logs faltante

Si el Dashboard no arranca, crea el directorio de logs:

```bash
sudo mkdir -p /var/log/wazuh-dashboard
sudo chown -R wazuh-dashboard:wazuh-dashboard /var/log/wazuh-dashboard
sudo chmod 750 /var/log/wazuh-dashboard
sudo systemctl restart wazuh-dashboard
```

### Paso 7 — Inicializar el cluster de seguridad

```bash
sudo bash wazuh-install.sh -s
```

Debe mostrar:
```
INFO: Indexer cluster config initialized.
```

---

## Obtener las credenciales

```bash
sudo tar -O -xvf ~/wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Guarda bien estas contraseñas — especialmente las de `admin` y `wazuh-wui`.

---

## Configurar la conexión API del Dashboard

El Dashboard intenta conectar a `localhost` por defecto, lo que no funciona. Hay que apuntarlo a la IP real.

```bash
sudo nano /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

Edita la sección `hosts`:

```yaml
hosts:
  - default:
      url: https://192.168.64.6      # Tu IP
      port: 55000
      username: wazuh-wui
      password: TU_PASSWORD_WAZUH_WUI
      run_as: true
```

Guarda y reinicia:

```bash
sudo systemctl restart wazuh-dashboard
```

### Configurar credenciales del Dashboard hacia el Indexer

```bash
sudo grep -E "opensearch.password|opensearch.username" /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Si están comentadas, descoméntalas:

```bash
sudo sed -i 's/# opensearch.username: kibanaserver/opensearch.username: kibanaserver/' /etc/wazuh-dashboard/opensearch_dashboards.yml

sudo sed -i 's/# opensearch.password: kibanaserver/opensearch.password: TU_PASSWORD_KIBANASERVER/' /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Reinicia el Dashboard:

```bash
sudo systemctl restart wazuh-dashboard
```

---

## Habilitar arranque automático de servicios

Para que los servicios arranquen solos tras cada reinicio:

```bash
sudo systemctl enable wazuh-indexer wazuh-manager wazuh-dashboard
```

> ⚠️ En ARM64 el Indexer puede tardar más de lo que systemd espera por defecto. El timeout de 600 segundos configurado anteriormente garantiza que no se mate el proceso antes de que termine de arrancar.

---

## Verificación final

Comprueba que los tres servicios están corriendo:

```bash
sudo systemctl status wazuh-indexer wazuh-manager wazuh-dashboard | grep -E "Active|●"
```

Los tres deben mostrar `active (running)`.

Verifica que el Dashboard está escuchando:

```bash
sudo ss -tlnp | grep 443
```

Debe aparecer una línea con `node` escuchando en el puerto `443`.

---

## Acceso al Dashboard

Abre el navegador y ve a:

```
https://TU_IP
```

- **Usuario:** `admin`
- **Password:** la obtenida en el paso de credenciales

> ℹ️ El navegador mostrará un aviso de certificado no confiable — es normal, acepta la excepción.

---

## Comportamiento tras reinicios

El Wazuh Indexer tarda en arrancar en ARM64. Si tras un reinicio el Indexer no arranca automáticamente:

```bash
sudo systemctl start wazuh-indexer
# Espera 2-3 minutos
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
```

El timeout de 600 segundos configurado anteriormente persiste entre reinicios — no hace falta volver a configurarlo.

---

## Resumen de puertos

| Servicio | Puerto | Descripción |
|----------|--------|-------------|
| Wazuh Dashboard | 443 | Interfaz web |
| Wazuh Indexer | 9200 | API OpenSearch |
| Wazuh Manager API | 55000 | API REST |
| Wazuh Agent | 1514 | Comunicación agentes |
| Wazuh Enrollment | 1515 | Registro agentes |

---

## Próximos pasos

Una vez Wazuh está funcionando, el siguiente paso es conectar agentes para empezar a monitorizar:

1. **Agente en el propio Ubuntu** — ya monitorizado por defecto
2. **Agentes en contenedores Docker** — DVWA, WebGoat, JuiceShop
3. **Agente en Kali** — para monitorizar la máquina atacante

Con todos los agentes conectados puedes practicar escenarios reales de ataque y defensa (Red Team / Blue Team) y ver las alertas en tiempo real en el Dashboard.

---

*Guía elaborada tras instalación real en Mac M1 con UTM · Marzo 2026*
