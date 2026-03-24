# 🛡️ Informe de Sesión — Fases 1 y 2 del Lab de Ciberseguridad

**Fecha:** 24 de marzo de 2026  
**Entorno:** Mac M1 (host) → UTM → Ubuntu Virtualized ARM64 (192.168.64.6)  
**SIEM:** Wazuh 4.14.4  
**Máquina atacante:** Kali Linux (VM UTM)  
**Objetivo:** DVWA (Docker, 192.168.64.6:8081)

---

## 📋 Entorno del Lab

| Componente | Detalle |
|-----------|---------|
| Host | Mac M1 8GB RAM |
| Virtualización | UTM |
| VM principal | Ubuntu 25.10 ARM64 (192.168.64.6) |
| SIEM | Wazuh 4.14.4 (nativo ARM64) |
| Atacante | Kali Linux VM |
| Objetivos | DVWA (8081), WebGoat (8082), JuiceShop (3000) |
| Agentes Wazuh | ID 000 (servidor), ID 001 (DVWA), ID 002 (WebGoat) |

---

## ✅ Fase 1 — Reconocimiento y Fuerza Bruta

### 1.1 SSH Brute Force con Hydra

**Objetivo:** Detectar ataque de fuerza bruta SSH con Wazuh.

**Herramienta:** Hydra  
**Comando ejecutado desde Kali:**
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.64.6
```

**Resultado en Wazuh:**

| Campo | Valor |
|-------|-------|
| Regla | 5758 |
| Descripción | SSHD brute force trying to get access to the system |
| Nivel | Alto |
| Agente | edward-ubuntu (ID 000) |

**Conclusión:** Wazuh detectó el ataque correctamente leyendo `/var/log/auth.log` a través del agente local.

---

### 1.2 Escaneo de Puertos con Nmap

**Objetivo:** Detectar reconocimiento de red con Wazuh.

**Herramienta:** Nmap  
**Comando ejecutado desde Kali:**
```bash
nmap -sV 192.168.64.6
```

**Resultado en Wazuh:**

| Campo | Valor |
|-------|-------|
| Regla | 100100 |
| Descripción | iptables: Posible escaneo de puertos detectado |
| Nivel | 10 (Medium) |
| Agente | edward-ubuntu (ID 000) |

**Conclusión:** iptables registró el escaneo en el log del sistema. El agente Wazuh leyó ese log y el Manager aplicó la regla 100100. **Wazuh no analiza tráfico de red directamente — analiza logs.**

---

## ✅ Fase 2 — Ataques a Aplicación Web (DVWA)

### 2.1 SQL Injection Manual

**Objetivo:** Comprobar vulnerabilidad SQLi en DVWA nivel Low.

**Herramienta:** Navegador web  
**Payload introducido en DVWA → SQL Injection:**
```
1' OR '1'='1
```

**Query resultante en el servidor:**
```sql
SELECT * FROM users WHERE id = '1' OR '1'='1'
```

**Resultado:** DVWA devolvió todos los usuarios de la base de datos:
- admin / admin
- Gordon Brown
- Hack Me
- Pablo Picasso
- Bob Smith

**Detección en Wazuh:** ❌ No detectado  
**Motivo:** Una sola petición HTTP no genera volumen suficiente para disparar reglas de iptables. Wazuh no tiene acceso a los logs de Apache de DVWA por defecto.

---

### 2.2 SQL Injection Automatizada con sqlmap

**Objetivo:** Automatizar SQLi y comprobar si Wazuh detecta el volumen de tráfico.

**Herramienta:** sqlmap  
**Comando ejecutado desde Kali:**
```bash
sqlmap -u "http://192.168.64.6:8081/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=<session_id>; security=low" \
  --dbs --batch
```

**Resultado de sqlmap:**
```
available databases [2]:
[*] dvwa
[*] information_schema

Técnicas de inyección detectadas:
- boolean-based blind
- error-based (EXTRACTVALUE)
- time-based blind (SLEEP)
- UNION query (2 columnas)

Back-end DBMS: MySQL >= 5.1 (MariaDB fork)
Web server: Apache 2.4.66, PHP 8.5.3
```

**Detección en Wazuh:** ✅ Detectado

| Campo | Valor |
|-------|-------|
| Regla | 100100 |
| Descripción | iptables: Posible escaneo de puertos detectado |
| Nivel | 10 (Medium) |

**Motivo de detección:** sqlmap generó 146 peticiones HTTP en segundos, lo que iptables interpretó como comportamiento anómalo (similar a un escaneo de puertos).

> ⚠️ **Importante:** Wazuh detectó el *volumen* de tráfico, no el *contenido* de la inyección SQL. No distingue entre un sqlmap y un Nmap — ambos disparan la regla 100100.

---

### 2.3 XSS Reflected Manual

**Objetivo:** Explotar vulnerabilidad XSS Reflected en DVWA.

**Herramienta:** Navegador web  
**Payload introducido en DVWA → XSS (Reflected):**
```html
<script>alert('XSS')</script>
```

**Resultado:** DVWA ejecutó el script y mostró el popup `alert('XSS')` — vulnerabilidad confirmada.

**Detección en Wazuh:** ❌ No detectado  
**Motivo:** Una sola petición HTTP sin volumen anómalo. Wazuh no analiza el payload de las peticiones web sin logs de aplicación o WAF.

---

## 📊 Resumen de Detecciones

| Fase | Ataque | Herramienta | Wazuh | Regla | Método de detección |
|------|--------|-------------|-------|-------|---------------------|
| 1 | SSH Brute Force | Hydra | ✅ | 5758 | auth.log |
| 1 | Port Scan | Nmap | ✅ | 100100 | iptables log |
| 2 | SQLi manual | Navegador | ❌ | - | Sin logs Apache |
| 2 | SQLi automatizada | sqlmap | ✅ | 100100 | iptables (volumen) |
| 2 | XSS Reflected | Navegador | ❌ | - | Sin logs Apache |

---

## 🧠 Lecciones Aprendidas

### Wazuh es un analizador de logs, no un sniffer
Wazuh no captura tráfico de red. Detecta amenazas leyendo logs del sistema operativo (`auth.log`, logs de iptables, etc.). Para detectar ataques web necesita acceso a logs de aplicación (Apache, Nginx) o integración con un WAF.

### La detección depende del volumen, no del contenido
Ataques manuales lentos (SQLi manual, XSS manual) son invisibles para Wazuh sin configuración adicional. Herramientas automatizadas (sqlmap, Nmap, Hydra) generan patrones anómalos que iptables registra.

### Cadena de detección completa
```
Ataque → Log del sistema → Agente Wazuh → Manager → Filebeat → Indexer → Dashboard
```

---

## 🔧 Pendientes

- [ ] Reinstalar OpenVAS correctamente (instalación previa incompleta)
- [ ] Configurar logs de Apache en `ossec.conf` para detectar SQLi/XSS por contenido
- [ ] Fase 3: Prometheus + Node Exporter + Grafana
- [ ] Guía Markdown instalación Wazuh ARM64 para GitHub

---

## 🖥️ Infraestructura de referencia

```
Mac M1 (host)
└── UTM
    ├── Ubuntu Virtualized ARM64 (192.168.64.6)
    │   ├── Wazuh 4.14.4 (indexer + manager + dashboard)
    │   ├── Filebeat
    │   └── Docker
    │       ├── lab-seguridad-dvwa-1 (8081) — Agente Wazuh ID 001
    │       ├── lab-seguridad-webgoat-1 (8082) — Agente Wazuh ID 002
    │       ├── lab-seguridad-juice-shop-1 (3000)
    │       ├── lab-seguridad-db-1 (MariaDB 3306)
    │       └── nagios (8080)
    └── Kali Linux VM (atacante)
```
