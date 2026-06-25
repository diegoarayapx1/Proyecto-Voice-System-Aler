# Proyecto-Voice-System-Aler
# 🔔 VoiceAlert System

> Sistema de alertas multi-canal con escalamiento a llamada de voz vía Asterisk/FreePBX, orquestado por n8n y desplegado en AWS EC2.

![Architecture](architecture/diagram.png)

---

## 📋 Descripción

VoiceAlert es un sistema de alertas automatizadas diseñado para entornos donde una notificación crítica debe llegar **sí o sí** al personal de guardia. Combina automatización de workflows (n8n), mensajería (Slack, Telegram) y telefonía IP (Asterisk/FreePBX) en un flujo de escalamiento progresivo:

1. Se detecta un evento crítico (palabra clave en Slack o alerta de Zabbix)
2. El sistema verifica que quien dispara la alerta tiene el rol autorizado
3. Notifica en paralelo por Slack y Telegram
4. Si el evento lo requiere, escala a **llamada de voz** a los guardias de turno

Este proyecto integra conceptos de **redes**, **seguridad**, **automatización** y **observabilidad** en una solución funcional sobre infraestructura cloud real.

---

## 🏗️ Arquitectura

```
[Slack #alertas]──────────────────────┐
                                      ▼
[Zabbix webhook] ──────────► [EC2 #1 — n8n]
                               │  Docker + Nginx + SSL
                               │  Control de acceso (rol)
                               │  Logging de alertas
                               │
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
           [Slack notif]  [Telegram bot]  [Llamada voz]
                                               │
                                          [EC2 #2 — Asterisk]
                                           gTTS + ffmpeg
                                           Fail2ban (SSH+SIP)
                                               │
                                    ┌──────────┴──────────┐
                                    ▼                     ▼
                               [Anexo 7001]          [Anexo 7002]
                             Guardia primario     Guardia secundario
```

### Componentes principales

| Componente | Tecnología | Rol |
|---|---|---|
| Orquestador | n8n (Docker) | Lógica de flujo y triggers |
| Web proxy | Nginx + SSL (Let's Encrypt) | Exposición segura de n8n |
| PBX | Asterisk / FreePBX | Llamadas SIP |
| Voz sintética | gTTS + ffmpeg | Generación de audio |
| Monitoreo | Zabbix | Fuente de alertas de infraestructura |
| Protección | Fail2ban | Hardening SSH y SIP |
| Backup | Bash + cron | Respaldo de configuración Asterisk |

---

## 📁 Estructura del repositorio

```
voicealert-system/
│
├── README.md
├── architecture/
│   └── diagram.png            # Diagrama de arquitectura y red
│
├── n8n/
│   └── workflow.json          # Workflow exportado desde n8n
│
├── asterisk/
│   ├── extensions.conf        # Dialplan de alertas
│   └── sip.conf               # Configuración de anexos SIP
│
├── scripts/
│   ├── generate_voice.py      # Generación de audio con gTTS + ffmpeg
│   └── backup_asterisk.sh     # Backup automático a S3 / local
│
├── security/
│   ├── fail2ban_sip.conf      # Regla Fail2ban para SIP
│   └── security_groups.md     # Puertos habilitados por instancia
│
└── docs/
    ├── setup_ec2.md           # Guía de despliegue paso a paso
    ├── n8n_config.md          # Configuración de variables y credenciales
    └── future_improvements.md # Mejoras documentadas (no implementadas)
```

---

## 🚀 Despliegue rápido

### Requisitos previos

- Cuenta AWS (se usaron instancias t3.micro dentro del Free Tier)
- Docker y Docker Compose instalados en EC2 #1
- Python 3.8+ en EC2 #2
- Dominio o IP pública para n8n (ver `docs/setup_ec2.md`)

### 1. Levantar EC2 #1 — n8n

```bash
# Clonar repositorio
git clone https://github.com/tu-usuario/voicealert-system.git
cd voicealert-system

# Levantar n8n con Docker
docker compose up -d

# Verificar
docker ps
```

### 2. Importar workflow en n8n

1. Abrir n8n en `https://tu-ip:5678`
2. Ir a **Workflows → Import from file**
3. Seleccionar `n8n/workflow.json`
4. Configurar credenciales de Slack y Telegram en **Credentials**

### 3. Levantar EC2 #2 — Asterisk

```bash
# Instalar dependencias
sudo apt update && sudo apt install -y asterisk python3-pip ffmpeg
pip3 install gTTS

# Copiar configuración
sudo cp asterisk/extensions.conf /etc/asterisk/
sudo cp asterisk/sip.conf /etc/asterisk/
sudo asterisk -rx "core reload"
```

Ver guía completa en [`docs/setup_ec2.md`](docs/setup_ec2.md).

---

## 🔐 Seguridad implementada

### Fail2ban
Protección contra fuerza bruta en SSH y protocolo SIP en la instancia Asterisk. Configuración en `security/fail2ban_sip.conf`.

### Security Groups AWS
Política de mínimo privilegio — solo puertos estrictamente necesarios:

| Instancia | Puerto | Protocolo | Origen |
|---|---|---|---|
| EC2 #1 (n8n) | 443 | HTTPS | 0.0.0.0/0 |
| EC2 #1 (n8n) | 22 | SSH | IP admin |
| EC2 #2 (Asterisk) | 5060 | SIP/UDP | EC2 #1 |
| EC2 #2 (Asterisk) | 10000-20000 | RTP/UDP | EC2 #1 |
| EC2 #2 (Asterisk) | 22 | SSH | IP admin |

Ver detalle en [`security/security_groups.md`](security/security_groups.md).

### Control de acceso
Verificación de rol **Líder de Grupo** antes de ejecutar cualquier alerta. Lista de autorizados gestionada en el workflow de n8n (hardcodeada en esta versión — ver mejoras futuras).

### Observabilidad
Registro de cada evento: quién disparó la alerta, cuándo, si fue autorizada o rechazada. Log persistente accesible desde n8n.

---

## 🎯 Decisiones técnicas documentadas

### ¿Por qué gTTS en lugar de Gemini TTS?
gTTS es gratuito, no requiere API key, y la latencia es aceptable para alertas no críticas en tiempo real. Gemini TTS ofrecería mejor calidad de voz y menor latencia, pero agrega dependencia externa y costo por llamada. Para el alcance de este proyecto, gTTS + ffmpeg cumple el objetivo.

### ¿Por qué n8n y no AWS Lambda o un script Python?
n8n permite modificar la lógica del flujo visualmente sin redesplegar código, lo que facilita mantenimiento. El workflow exportado en JSON es versionable en Git igual que cualquier script.

### ¿Por qué no IPs elásticas?
Decisión de costo: las IPs elásticas tienen costo cuando no están asociadas a una instancia en ejecución. Para un entorno de demostración / portafolio, se documenta como mejora de producción.

---

## 🔮 Mejoras futuras documentadas

Ver detalle completo en [`docs/future_improvements.md`](docs/future_improvements.md).

- **VPN site-to-site IPsec** entre EC2 #1 y EC2 #2 en lugar de exposición directa al internet *(referencia directa a configuración CCNA Security — OSPF MD5, ZBF, IPsec)*
- **IPs elásticas** para ambas instancias en entorno de producción
- **Integración con IdP** (Okta, Azure AD) para gestión de roles en lugar de lista hardcodeada
- **Base de datos de autorizados** (PostgreSQL / DynamoDB) para escalar el control de acceso
- **Alta disponibilidad** de n8n con réplica activa-pasiva

---

## 📸 Demo

> *GIF / capturas del flujo funcionando — se agregarán al completar el despliegue en AWS.*

---

## 🛠️ Stack tecnológico

![AWS](https://img.shields.io/badge/AWS-EC2-orange?logo=amazon-aws)
![Docker](https://img.shields.io/badge/Docker-Compose-blue?logo=docker)
![n8n](https://img.shields.io/badge/n8n-workflow-red)
![Asterisk](https://img.shields.io/badge/Asterisk-FreePBX-orange)
![Python](https://img.shields.io/badge/Python-gTTS-blue?logo=python)
![Nginx](https://img.shields.io/badge/Nginx-SSL-green?logo=nginx)
![Zabbix](https://img.shields.io/badge/Zabbix-monitoring-red)

---

## 👤 Autor

**Diego Araya** — Estudiante de Ingeniería en Conectividad y Redes, Duoc UC  
Área de interés: SOC Analysis · Network Security · GRC  
[![LinkedIn](https://img.shields.io/badge/LinkedIn-perfil-blue?logo=linkedin)](https://linkedin.com/in/tu-perfil)
[![GitHub](https://img.shields.io/badge/GitHub-portafolio-black?logo=github)](https://github.com/tu-usuario)

---

## 📄 Licencia

MIT License — libre para usar, modificar y distribuir con atribución.
