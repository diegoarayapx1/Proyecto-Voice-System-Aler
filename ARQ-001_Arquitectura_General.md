# ARQ-001 · Arquitectura General — VoiceAlert

**Versión:** 3.0  
**Estado:** Definitivo  
**Proyecto:** VoiceAlert — Sistema de alertas automatizadas con escalamiento progresivo  
**Curso:** CUY5132 Comunicaciones Unificadas · DUOC UC

---

## 1. Descripción del sistema

VoiceAlert es un sistema de alertas críticas con enfoque SOC que garantiza que una notificación llegue **sí o sí** al personal de guardia mediante escalamiento progresivo y canales diferenciados por rol. Combina automatización de workflows (n8n), mensajería multi-canal (Slack, Telegram, WhatsApp), telefonía IP (FreePBX/Asterisk), monitoreo de infraestructura (Zabbix), SIEM (Wazuh) y hardening de seguridad sobre infraestructura cloud AWS con RHEL 9.

---

## 2. Sistema de roles

### 2.1 Definición de roles y canales

| Rol | Puede contactar | Keyword `URGENTE` | Canal alerta | Activa llamada |
|---|---|---|---|---|
| **Trabajador** | Líder / IT (ticket) | Sin efecto extra | Slack + copia Líder | No |
| **Líder** | IT | Activa flujo completo | WhatsApp (CallMeBot) | Sí |
| **IT** | — | — | Receptor final | — |

### 2.2 Reglas de escalamiento

| Acción | Resultado |
|---|---|
| Trabajador → IT (mensaje) | Ticket simple por Slack + copia al Líder |
| Trabajador → IT (intenta llamar) | Denegado + log de intento |
| Trabajador → IT (`URGENTE`) | Mismo ticket, keyword ignorada para llamada |
| Líder → IT sin `URGENTE` | Mensaje por WhatsApp a IT |
| Líder → IT con `URGENTE` | WhatsApp + llamada encadenada 7001→7002→7003 |
| Zabbix webhook | Máxima prioridad → todos los canales + llamada |
| Cualquier acción | Log JSON registrado en Wazuh vía syslog |

### 2.3 Escalamiento de llamadas (fallback)

Si el guardia primario no contesta en 20 segundos, Asterisk intenta el siguiente anexo automáticamente:

```
Llama 7001 → sin respuesta 20s
      → Llama 7002 → sin respuesta 20s
            → Llama 7003 → sin respuesta 20s
                  → Log "sin respuesta en todos los anexos"
```

Implementado con `RetryTime` y `MaxRetries` en call file de Asterisk.

### 2.4 Implementación de roles

Roles implementados como **JSON estático en variables de entorno de n8n**. Mejora futura documentada: integración con IdP (FreeIPA, Okta) o base de datos.

```json
{
  "U00000001": { "nombre": "Trabajador 1",   "rol": "trabajador", "anexo": "7005" },
  "U00000002": { "nombre": "Líder",          "rol": "lider",      "anexo": "7004" },
  "U00000003": { "nombre": "IT Guardia 1",   "rol": "IT",         "anexo": "7001" },
  "U00000004": { "nombre": "IT Guardia 2",   "rol": "IT",         "anexo": "7002" },
  "U00000005": { "nombre": "IT Guardia 3",   "rol": "IT",         "anexo": "7003" }
}
```

---

## 3. Usuarios y extensiones

### 3.1 Cuentas Slack (workspace propio)

| Navegador | Email | Rol | Extensión SIP |
|---|---|---|---|
| Chrome (principal) | trabajador1.voicealert@gmail.com | Trabajador | 7005 |
| Firefox | lider.voicealert@gmail.com | Líder | 7004 |
| Edge | it1.voicealert@gmail.com | IT / Guardia 1 | 7001 |
| Chrome (perfil 2) | it2.voicealert@gmail.com | IT / Guardia 2 | 7002 |
| Firefox (privado) | it3.voicealert@gmail.com | IT / Guardia 3 | 7003 |

### 3.2 Extensiones SIP (carga masiva CSV en FreePBX)

| Extensión | Rol | Softphone | Estado |
|---|---|---|---|
| 7001 | IT Guardia primario | MicroSIP (Windows) | Activo |
| 7002 | IT Guardia secundario | 3CXPhone (Windows) | Activo |
| 7003 | IT Guardia terciario | Linphone (Windows) | Activo |
| 7004 | Líder | — | Activo |
| 7005 | Trabajador | — | Activo |
| 7100 | Extensión de prueba | — | Deshabilitado |

> Las extensiones se provisionan mediante **carga masiva CSV** desde el panel de FreePBX, demostrando aprovisionamiento empresarial de extensiones.

---

## 4. Infraestructura AWS

### 4.1 Resumen de instancias

| Instancia | Rol | SO | Tipo | Disco | IP |
|---|---|---|---|---|---|
| EC2-1 | n8n + Zabbix | RHEL 9 | t3.large | 30 GiB | Elástica |
| EC2-2 | FreePBX / Asterisk | RHEL 9 | t3.large | 30 GiB | Elástica |
| EC2-3 | Wazuh SIEM | RHEL 9 | t3.large | 50 GiB | Elástica |

### 4.2 EC2-1 — Automatización

| Parámetro | Valor |
|---|---|
| SO | RHEL 9 |
| Servicios | n8n (Podman/Quadlet), Nginx, Certbot SSL, Zabbix (Podman) |
| Puertos SG | 22, 80, 443, 5678, 10051 (Zabbix) |
| Logs | rsyslog → EC2-3 Wazuh |

### 4.3 EC2-2 — Telefonía

| Parámetro | Valor |
|---|---|
| SO | RHEL 9 |
| Servicios | FreePBX/Asterisk (Podman/Quadlet), gTTS, ffmpeg, Fail2ban |
| Puertos SG | 22, 80, 443, 5060/UDP (SIP), 10000-20000/UDP (RTP) |
| Logs | rsyslog → EC2-3 Wazuh |

### 4.4 EC2-3 — SIEM y monitoreo

| Parámetro | Valor |
|---|---|
| SO | RHEL 9 |
| Servicios | Wazuh Manager + Dashboard (Podman/Quadlet) |
| Firewall | AWS Security Group + **firewalld** |
| Puertos SG | 22, 443 (dashboard), 514/UDP (syslog), 1514, 1515 (Wazuh) |
| Recibe logs de | EC2-1 y EC2-2 vía rsyslog |

### 4.5 Estimación de costos (72h uptime)

| Recurso | Costo/hora | 72h |
|---|---|---|
| EC2-1 t3.large | $0.0832 | ~$5.99 |
| EC2-2 t3.large | $0.0832 | ~$5.99 |
| EC2-3 t3.large | $0.0832 | ~$5.99 |
| IPs elásticas x3 | $0 (asociadas) | $0 |
| **Total** | | **~$17.97 USD** |

---

## 5. Stack tecnológico completo

| Capa | Tecnología | Instancia | Justificación |
|---|---|---|---|
| Automatización | n8n (Podman/Quadlet) | EC2-1 | Workflows visuales, SSH nativo |
| IA | OpenRouter · Mistral 7B free | EC2-1 (via API) | Sin costo, clasificación de severidad |
| Trigger mensajería | Slack (workspace propio) | Externo | Multi-cuenta, bot gratuito |
| Notificación estándar | Telegram Bot API | EC2-1 (via API) | Gratuito, HTTP simple |
| Notificación crítica | WhatsApp vía CallMeBot | EC2-1 (via API) | Gratuito, HTTP Request |
| Canal auditoría | Slack #auditoria | Externo | Log visible para todos |
| Síntesis de voz | gTTS (Python) | EC2-2 | Sin API key, validado en lab |
| Conversión audio | ffmpeg | EC2-2 | WAV → SLIN 8kHz Asterisk |
| Telefonía | FreePBX / Asterisk (Podman) | EC2-2 | IVR, call files, SIP |
| Monitoreo infra | Zabbix (Podman) | EC2-1 | Genera alertas reales de las EC2 |
| SIEM | Wazuh (Podman) | EC2-3 | Centraliza logs, dashboard SOC |
| Logs sistema | rsyslog | EC2-1, EC2-2 | Reenvío syslog a Wazuh |
| Protección SSH/SIP | Fail2ban | EC2-2, EC2-3 | Bloqueo fuerza bruta |
| Firewall adicional | firewalld | EC2-3 | Hardening RHEL 9 nativo |
| Contenedores | Podman + Quadlet | Todas | Nativo RHEL 9, systemd units |
| DNS dinámico | No-IP DDNS | Todas | Apunta a IPs elásticas |
| SSL | Let's Encrypt / Certbot | EC2-1 | HTTPS para n8n |
| Backup | Script + systemd timer | EC2-2 | Dialplan + configs Asterisk |

---

## 6. Flujo completo de escalamiento

```
Evento detectado
(Slack app_mention / Zabbix webhook / HTTP manual)
        │
        ▼
Identificar usuario + rol
(JSON en variables de entorno n8n)
        │
   ┌────┴──────────────────────┐
   │                           │
Sin permiso               Con permiso
(ej: Trabajador→IT llamada)    │
   │                           ▼
   ▼                    Agente IA clasifica
Respuesta denegada       severidad + contexto
+ log syslog → Wazuh          │
                    ┌──────────┴──────────┐
                    │                     │
             Trabajador→IT           Líder→IT
                    │                     │
                    ▼                     ▼
           Ticket Slack          ¿Contiene URGENTE?
           + copia Líder                  │
           + log → Wazuh         ┌────────┴────────┐
                                 │No               │Sí
                                 ▼                 ▼
                           WhatsApp IT       WhatsApp IT
                           + log → Wazuh     + llamada encadenada
                                             7001→7002→7003
                                             + log → Wazuh
                                             + #auditoria Slack
```

---

## 7. Zabbix en el proyecto

Zabbix corre como contenedor Podman en EC2-1 y cumple dos roles:

**Monitoreo real:** Supervisa las tres EC2 (CPU, RAM, disco, servicios). Si FreePBX cae, Zabbix lo detecta y dispara un webhook a n8n que recorre el flujo completo con máxima prioridad.

**Simulación de alertas:** Permite crear triggers manuales para demostrar el flujo sin romper infraestructura real. Ejemplo: trigger "servidor caído" → webhook → n8n → WhatsApp + llamada.

---

## 8. Observabilidad SOC

### 8.1 Estructura de log por evento

```json
{
  "timestamp": "2025-01-15T03:22:11Z",
  "user_id": "U00000002",
  "nombre": "Líder",
  "rol": "lider",
  "mensaje": "Servidor caído URGENTE",
  "autorizado": true,
  "canal_notificacion": "whatsapp",
  "llamada_originada": true,
  "anexos_intentados": ["7001", "7002"],
  "anexo_contestado": "7002",
  "severidad_ia": "critica"
}
```

### 8.2 Canales de auditoría

| Canal | Contenido | Audiencia |
|---|---|---|
| Slack `#auditoria` | Toda acción del sistema | Solo lectura, todos los roles |
| Wazuh dashboard | Logs consolidados EC2-1 y EC2-2 | Administrador |
| Log JSON local | Respaldo en EC2-1 | Administrador |

### 8.3 Seguridad

| Medida | Dónde | Qué protege |
|---|---|---|
| Fail2ban SSH | EC2-2, EC2-3 | Fuerza bruta SSH |
| Fail2ban SIP | EC2-2 | Escaneo de extensiones, flood REGISTER |
| firewalld | EC2-3 | Hardening RHEL 9, mínimo privilegio |
| AWS Security Groups | Todas | Exposición de puertos mínima |
| Roles JSON n8n | EC2-1 | Control de acceso por usuario |

---

## 9. Arquitectura recomendada (documentada, no implementada)

**VPN site-to-site IPsec entre las tres instancias.** La comunicación SSH entre EC2-1 y EC2-2, y el syslog hacia EC2-3, quedarían dentro del túnel VPN eliminando exposición a internet público. Referencia: IKEv2, IPsec tunnel mode (CCNA Security).

**Mejora futura — escalamiento automático por tiempo:** si un ticket de Trabajador no recibe respuesta del Líder en X minutos, n8n escala automáticamente. Requiere workflow asincrónico con timer en n8n.

**Mejora futura — integración IdP:** reemplazar JSON estático de roles por FreeIPA o Okta para gestión de identidad empresarial real.

---

## 10. Decisiones de diseño documentadas

| Decisión | Alternativa descartada | Razón |
|---|---|---|
| gTTS en lugar de Gemini TTS | Gemini TTS (Google AI Studio) | Error 403 por billing en cuenta institucional |
| Podman en lugar de Docker | Docker en RHEL 9 | Podman es nativo RHEL 9, sin demonio root |
| CallMeBot para WhatsApp | WhatsApp Business API oficial | Requiere pago, CallMeBot es gratuito |
| JSON estático para roles | Base de datos / FreeIPA | Simplicidad para demo, documentado como mejora |
| Syslog en lugar de agente Wazuh | Agente Wazuh en cada EC2 | Limitación de recursos en AWS Student |
| IP elástica en lugar de dinámica | No-IP con IP dinámica | SIP/RTP requiere IP estable |

---

## 11. Entregables del repositorio

| Entregable | Descripción |
|---|---|
| `README.md` | Diagrama arquitectura, red, instrucciones despliegue |
| `workflows/voicealert.json` | Workflow n8n exportado |
| `scripts/gtts_call.sh` | Síntesis de voz + conversión ffmpeg |
| `scripts/backup_asterisk.sh` | Respaldo con systemd timer |
| `config/fail2ban/` | Filtros y jails SSH + SIP |
| `config/dialplan/` | Contexto `tts-saliente` + fallback 7001→7002→7003 |
| `config/extensiones.csv` | Carga masiva de extensiones FreePBX |
| `config/zabbix/` | Templates y triggers de alerta |
| `docs/` | ARQ-001, PLAN-001, NOD-001 a NOD-007, INF-001 a INF-003 |
| `assets/` | GIFs y capturas del flujo funcionando |

---

## 12. Documentos del proyecto

| ID | Tipo | Descripción |
|---|---|---|
| `ARQ-001` | Arquitectura | Este documento |
| `PLAN-001` | Timeline | Fases de implementación con checkpoints |
| `INF-001` | Infra | EC2-1: RHEL 9, Podman, n8n, Nginx, SSL, Zabbix |
| `INF-002` | Infra | EC2-2: RHEL 9, Podman, FreePBX, gTTS, Fail2ban |
| `INF-003` | Infra | EC2-3: RHEL 9, Podman, Wazuh, firewalld |
| `NOD-001` | Nodo | Slack Trigger: scopes, app_mention, workspace |
| `NOD-002` | Nodo | Verificación de roles: JSON, 3 tiers, denegado, log |
| `NOD-003` | Nodo | Agente IA: OpenRouter, prompt SOC, clasificación |
| `NOD-004` | Nodo | Notificaciones: Telegram + WhatsApp CallMeBot + Slack |
| `NOD-005` | Nodo | Condición escalamiento: URGENTE + rol Líder |
| `NOD-006` | Nodo | SSH gTTS: síntesis, ffmpeg, SLIN 8kHz |
| `NOD-007` | Nodo | SSH Call file: spool, fallback 7001→7002→7003 |

---

*Próximo documento: `PLAN-001` — Timeline de implementación*
