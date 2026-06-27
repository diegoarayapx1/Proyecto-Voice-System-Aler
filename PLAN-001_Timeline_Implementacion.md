# PLAN-001 · Timeline de Implementación — VoiceAlert

**Versión:** 1.0  
**Estado:** Definitivo  
**Proyecto:** VoiceAlert — Sistema de alertas automatizadas con escalamiento progresivo  
**Curso:** CUY5132 Comunicaciones Unificadas · DUOC UC

---

## Principio de construcción

Cada fase debe ser **funcional antes de avanzar a la siguiente**. Si un checkpoint falla, no se continúa. Esto garantiza que los problemas se aíslen en la fase correcta y no se arrastren al siguiente paso.

```
FASE 1 → FASE 2 → FASE 3 → FASE 4 → FASE 5 → FASE 6 → FASE 7
Infra     EC2-1    EC2-2    EC2-3    Flujo    Telefonía  Integración
base      n8n      FreePBX  Wazuh    n8n      voz        final
```

**Tiempo total estimado:** 17–24h de trabajo efectivo  
**Uptime máximo recomendado:** 72h (dentro de presupuesto AWS Student)

---

## FASE 1 — Infraestructura base
**Duración estimada:** 2–3h  
**Referencia:** `INF-001`, `INF-002`, `INF-003`

### Pasos

| # | Paso | Descripción |
|---|---|---|
| 1.1 | Crear EC2-1 | RHEL 9, t3.large, 30 GiB, IP elástica |
| 1.2 | Crear EC2-2 | RHEL 9, t3.large, 30 GiB, IP elástica |
| 1.3 | Crear EC2-3 | RHEL 9, t3.large, 50 GiB, IP elástica |
| 1.4 | Configurar Security Groups | Puertos mínimos por instancia según ARQ-001 |
| 1.5 | Asociar IPs elásticas | Una por instancia |
| 1.6 | Registrar DNS en No-IP | n8n.ddns.net, freepbx.ddns.net, wazuh.ddns.net |
| 1.7 | Verificar acceso SSH | Conectar a las 3 instancias desde terminal local |
| 1.8 | Instalar Podman | En las 3 instancias (nativo RHEL 9) |
| 1.9 | Configurar rsyslog | EC2-1 y EC2-2 apuntan a EC2-3 puerto 514/UDP |

**✅ Checkpoint FASE 1:** SSH funcional en las 3 EC2. rsyslog enviando logs de prueba a EC2-3.

---

## FASE 2 — EC2-1: n8n operativo
**Duración estimada:** 2–3h  
**Referencia:** `INF-001`

### Pasos

| # | Paso | Descripción |
|---|---|---|
| 2.1 | Crear Quadlet para n8n | Unit file systemd para Podman |
| 2.2 | Configurar variables de entorno | DOMAIN, SSL, credenciales admin |
| 2.3 | Instalar Nginx | Reverse proxy hacia n8n puerto 5678 |
| 2.4 | Configurar Certbot SSL | Certificado Let's Encrypt para el dominio |
| 2.5 | Habilitar servicio systemd | `systemctl enable --now n8n` vía Quadlet |
| 2.6 | Acceder al panel n8n | Verificar HTTPS desde browser |
| 2.7 | Crear usuario administrador | Configurar cuenta admin en n8n |
| 2.8 | Desplegar Zabbix en Podman | Contenedor Zabbix + base de datos |
| 2.9 | Configurar Zabbix | Agregar hosts EC2-1, EC2-2, EC2-3 para monitoreo |

**✅ Checkpoint FASE 2:** Panel n8n accesible por HTTPS. Zabbix dashboard mostrando las 3 EC2.

---

## FASE 3 — EC2-2: FreePBX operativo
**Duración estimada:** 3–4h  
**Referencia:** `INF-002`

### Pasos

| # | Paso | Descripción |
|---|---|---|
| 3.1 | Intentar imagen FreePBX oficial | Buscar imagen Podman/Docker oficial de FreePBX |
| 3.2 | Fallback si falla | Usar `steph576/issabel-pbx` si oficial no funciona |
| 3.3 | Crear Quadlet para FreePBX | Unit file systemd para Podman |
| 3.4 | Configurar External Address | IP elástica EC2-2 en Asterisk SIP Settings |
| 3.5 | Instalar gTTS y ffmpeg | En EC2-2, fuera del contenedor |
| 3.6 | Crear carpeta custom Asterisk | `/var/lib/asterisk/sounds/custom/` |
| 3.7 | Configurar dialplan tts-saliente | Contexto para reproducir audio generado |
| 3.8 | Configurar dialplan fallback | Encadenamiento 7001→7002→7003 con RetryTime |
| 3.9 | Carga masiva extensiones CSV | Provisionar 7001–7005 + 7100 desde panel FreePBX |
| 3.10 | Configurar MicroSIP (7001) | Softphone Windows, extensión guardia primario |
| 3.11 | Configurar 3CXPhone (7002) | Softphone Windows, extensión guardia secundario |
| 3.12 | Configurar Linphone (7003) | Softphone Windows, extensión guardia terciario |
| 3.13 | Instalar Fail2ban | Filtros SSH y SIP |

**✅ Checkpoint FASE 3:** Llamada de prueba exitosa entre MicroSIP (7001) y 3CXPhone (7002). Fail2ban activo.

---

## FASE 4 — EC2-3: Wazuh operativo
**Duración estimada:** 3–4h  
**Referencia:** `INF-003`

### Pasos

| # | Paso | Descripción |
|---|---|---|
| 4.1 | Configurar firewalld | Puertos Wazuh: 514, 1514, 1515, 443 |
| 4.2 | Desplegar Wazuh en Podman | Wazuh Manager + Indexer + Dashboard |
| 4.3 | Crear Quadlet para Wazuh | Unit file systemd |
| 4.4 | Acceder a dashboard Wazuh | Verificar HTTPS desde browser |
| 4.5 | Verificar recepción syslog | Confirmar logs de EC2-1 y EC2-2 en dashboard |
| 4.6 | Crear reglas de alerta | Alertas para intentos denegados y llamadas |

**✅ Checkpoint FASE 4:** Dashboard Wazuh mostrando logs de EC2-1 y EC2-2 en tiempo real.

---

## FASE 5 — Flujo n8n: roles y mensajería
**Duración estimada:** 4–5h  
**Referencia:** `NOD-001` a `NOD-005`

### Pasos

| # | Paso | Nodo | Descripción |
|---|---|---|---|
| 5.1 | Crear workspace Slack | — | Workspace propio + 5 cuentas en distintos browsers |
| 5.2 | Crear Slack App | — | Bot con scopes necesarios, instalar en workspace |
| 5.3 | Configurar Slack Trigger | NOD-001 | app_mention en canal #alertas |
| 5.4 | Crear canal #auditoria | — | Solo lectura para todos los roles |
| 5.5 | Configurar roles JSON | NOD-002 | Variables de entorno n8n con los 5 usuarios |
| 5.6 | Nodo verificación de rol | NOD-002 | Lógica 3 tiers + mensaje denegado + log |
| 5.7 | Configurar Agente IA | NOD-003 | OpenRouter API key, Mistral 7B, prompt SOC |
| 5.8 | Crear bot Telegram | NOD-004 | BotFather, token, grupo guardia |
| 5.9 | Configurar CallMeBot | NOD-004 | Activar WhatsApp con número IT |
| 5.10 | Nodo notificaciones | NOD-004 | Telegram (Trabajador) + WhatsApp (Líder) |
| 5.11 | Nodo condición URGENTE | NOD-005 | Switch: URGENTE + rol Líder → rama llamada |
| 5.12 | Nodo ticket con copia | NOD-004 | Trabajador→IT genera ticket Slack + copia Líder |
| 5.13 | Probar flujo completo | — | Los 5 roles desde los 5 browsers |

**✅ Checkpoint FASE 5:** Mensaje de Trabajador genera ticket con copia al Líder. Mensaje de Líder con URGENTE genera WhatsApp a IT. Intentos sin permiso son denegados y logueados.

---

## FASE 6 — Integración telefonía
**Duración estimada:** 2–3h  
**Referencia:** `NOD-006`, `NOD-007`

### Pasos

| # | Paso | Nodo | Descripción |
|---|---|---|---|
| 6.1 | Configurar credencial SSH | — | Par de llaves RSA EC2-1 → EC2-2 en n8n |
| 6.2 | Nodo SSH gTTS | NOD-006 | Genera audio WAV + convierte SLIN 8kHz |
| 6.3 | Nodo SSH call file | NOD-007 | Crea call file + mueve al spool Asterisk |
| 6.4 | Conectar al flujo | — | Rama URGENTE → SSH gTTS → SSH call file |
| 6.5 | Probar llamada simple | — | Líder escribe URGENTE → llega llamada a 7001 |
| 6.6 | Probar fallback | — | 7001 no contesta → llama 7002 → llama 7003 |
| 6.7 | Verificar audio | — | Confirmar que el mensaje de voz se escucha claro |

**✅ Checkpoint FASE 6:** Mensaje de Líder con URGENTE genera llamada real con audio. Fallback 7001→7002→7003 funciona correctamente.

---

## FASE 7 — Integración final y hardening
**Duración estimada:** 2–3h  
**Referencia:** `INF-001`, `INF-002`, `INF-003`

### Pasos

| # | Paso | Descripción |
|---|---|---|
| 7.1 | Activar trigger Zabbix | Configurar webhook Zabbix → n8n con alerta real |
| 7.2 | Probar flujo Zabbix completo | Alerta Zabbix → todos los canales + llamada |
| 7.3 | Verificar logs en Wazuh | Confirmar que todos los eventos aparecen en dashboard |
| 7.4 | Configurar script backup | `backup_asterisk.sh` con systemd timer en EC2-2 |
| 7.5 | Revisar Security Groups | Auditar puertos abiertos, cerrar los innecesarios |
| 7.6 | Exportar workflow n8n | Guardar JSON del workflow completo |
| 7.7 | Capturar evidencias | GIFs y screenshots del flujo funcionando |
| 7.8 | Generar README final | Diagrama, instrucciones, tabla de roles |

**✅ Checkpoint FASE 7 (final):** Alerta Zabbix recorre el flujo de extremo a extremo. Logs visibles en Wazuh. Backup configurado. Evidencias capturadas.

---

## Resumen de tiempos

| Fase | Descripción | Tiempo estimado |
|---|---|---|
| FASE 1 | Infraestructura base | 2–3h |
| FASE 2 | EC2-1: n8n + Zabbix | 2–3h |
| FASE 3 | EC2-2: FreePBX | 3–4h |
| FASE 4 | EC2-3: Wazuh | 3–4h |
| FASE 5 | Flujo n8n completo | 4–5h |
| FASE 6 | Integración telefonía | 2–3h |
| FASE 7 | Integración final | 2–3h |
| **Total** | | **18–25h** |

---

## Orden de documentos por fase

| Fase | Documentos a consultar |
|---|---|
| FASE 1 | `ARQ-001`, `INF-001`, `INF-002`, `INF-003` |
| FASE 2 | `INF-001`, `NOD-001` (preparación Slack) |
| FASE 3 | `INF-002`, `NOD-006`, `NOD-007` |
| FASE 4 | `INF-003` |
| FASE 5 | `NOD-001`, `NOD-002`, `NOD-003`, `NOD-004`, `NOD-005` |
| FASE 6 | `NOD-006`, `NOD-007` |
| FASE 7 | Todos |

---

*Próximo documento: `INF-001` — Infraestructura EC2-1*
