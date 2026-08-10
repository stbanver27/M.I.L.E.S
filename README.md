<div align="center">

# 🕸️ Miles

### Asistente Virtual Híbrido (Edge/Nube) para el Hogar Inteligente

*"Con gran poder de automatización, viene una gran responsabilidad de privacidad."*

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Middleware-41BDF5?style=for-the-badge&logo=home-assistant&logoColor=white)](https://www.home-assistant.io/)
[![Docker](https://img.shields.io/badge/Docker-Contenedores-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![OpenCV](https://img.shields.io/badge/OpenCV-Visi%C3%B3n%20Artificial-5C3EE8?style=for-the-badge&logo=opencv&logoColor=white)](https://opencv.org/)
[![SQLite](https://img.shields.io/badge/SQLite-Base%20de%20Datos-003B57?style=for-the-badge&logo=sqlite&logoColor=white)](https://www.sqlite.org/)
[![Whisper](https://img.shields.io/badge/Whisper-STT-000000?style=for-the-badge&logo=openai&logoColor=white)](https://github.com/openai/whisper)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![License](https://img.shields.io/badge/Licencia-MIT-green?style=for-the-badge)](#)

</div>

---

## 📖 Descripción del Proyecto

**Miles** —en honor a Miles Morales el Spider-Man — es un asistente de hogar inteligente construido **desde cero**, sin depender de plataformas comerciales cerradas (Alexa, Google Home, Siri).

La motivación detrás de este proyecto es doble:

- **Privacidad y control total del dato:** el procesamiento sensible (activación por voz, biometría facial, presencia de usuarios) ocurre **localmente**, en un servidor físico dentro del propio hogar (*Edge Computing*), evitando el envío constante de audio o video a servidores de terceros.
- **Personalización real:** a diferencia de los asistentes genéricos, Miles mantiene una **base de conocimiento contextual** de quién está en casa, qué dispositivos están activos y el historial reciente, inyectando ese contexto en cada conversación con el LLM para lograr respuestas verdaderamente conscientes del entorno.

En lugar de una monolítica "caja negra", Miles está diseñado como un **ecosistema de microservicios desacoplados**, donde cada módulo (visión, audio, presencia, lógica conversacional) puede desarrollarse, probarse y escalarse de forma independiente.

---

## 🏗️ Arquitectura de Software

Miles sigue una arquitectura modular orientada a servicios, donde un **orquestador central** coordina módulos especializados que se comunican entre sí. Home Assistant actúa como capa de abstracción para los dispositivos físicos, evitando que la lógica de negocio dependa de protocolos particulares de cada fabricante.

```
                                ┌───────────────────────────┐
                                │      USUARIO (Voz)        │
                                └─────────────┬──────────────┘
                                              │
                     ┌────────────────────────▼────────────────────────┐
                     │           MÓDULO DE AUDIO (I/O)                 │
                     │  OpenWakeWord → Whisper (STT) → Edge-TTS (TTS)  │
                     └────────────────────────┬────────────────────────┘
                                              │ texto / intención
                                              ▼
     ┌───────────────────┐        ┌───────────────────────┐        ┌───────────────────┐
     │  MÓDULO DE VISIÓN │◄──────►│   NÚCLEO / ORQUESTADOR │◄──────►│  MÓDULO DE RED     │
     │  OpenCV + RTSP     │        │      (Python Core)     │        │  Escaneo Wi-Fi/MAC │
     │  face_recognition  │        │  Gestión de contexto    │        │  Presencia         │
     └─────────┬───────────┘        └───────────┬────────────┘        └─────────┬───────────┘
               │ identidad                       │ contexto/estado                │ presencia
               ▼                                  ▼                                ▼
     ┌────────────────────────────────────────────────────────────────────────────────┐
     │                   BASE DE DATOS LOCAL (SQLite) — Memoria de Miles              │
     │        Perfiles · Historial de eventos · Estado de dispositivos · Logs         │
     └───────────────────────────────────┬────────────────────────────────────────────┘
                                          │ contexto inyectado en el prompt
                                          ▼
                     ┌───────────────────────────────────────┐
                     │      MOTOR LÓGICO (LLM Reasoning)      │
                     │     API Gemini / OpenAI + Prompt        │
                     │     Engineering contextual              │
                     └────────────────────┬────────────────────┘
                                          │ acción / respuesta
                                          ▼
                     ┌───────────────────────────────────────┐
                     │     HOME ASSISTANT (Docker) — Middleware│
                     │   Estandarización de protocolos IoT      │
                     └────────────────────┬────────────────────┘
                                          │
                                          ▼
                        📺 TVs · 💡 Luces · 🔌 Enchufes · 🎥 Cámaras
```

**Principios de diseño aplicados:**

- **Separación de responsabilidades:** cada módulo (audio, visión, red, LLM) es un componente independiente con una única responsabilidad.
- **Bajo acoplamiento:** los módulos se comunican a través del orquestador central y la base de datos compartida, no directamente entre sí.
- **Home Assistant como capa de abstracción:** desacopla la lógica de Miles de los protocolos específicos de cada fabricante de dispositivos (ONVIF, MQTT, Zigbee, etc.).
- **Edge-first:** todo lo que puede procesarse localmente (wake word, STT, visión), se procesa localmente. La nube solo se usa para el razonamiento conversacional complejo (LLM).

---

## 🧰 Prerrequisitos

### Hardware

| Componente | Especificación mínima recomendada |
|---|---|
| Servidor central | Mini PC x86, Intel i5, 8GB RAM, SSD |
| Audio | Micrófono de campo lejano (ej. ReSpeaker USB Array) + altavoces |
| Cámaras | IP, compatibles con ONVIF/RTSP (ej. Yoosee), IP estática |
| Red | Conexión Ethernet dedicada para el servidor |

### Software

- Ubuntu 22.04 LTS o superior
- Python 3.10+
- Docker y Docker Compose
- `ffmpeg` (procesamiento de streams RTSP)
- Cuenta y API Key de Google Gemini y/u OpenAI
- Acceso administrativo al router (para escaneo de red / IPs estáticas)

---

## ⚙️ Instalación y Configuración Inicial

### 1. Clonar el repositorio

```bash
git clone https://github.com/tu-usuario/miles.git
cd miles
```

### 2. Configurar Home Assistant (Docker)

```bash
docker compose -f infra/homeassistant/docker-compose.yml up -d
```

Accede a `http://localhost:8123` para completar la configuración inicial y generar un **Long-Lived Access Token**, que Miles usará para comunicarse con la API de Home Assistant.

### 3. Crear entorno virtual e instalar dependencias de Python

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 4. Configurar variables de entorno

Copia el archivo de ejemplo y completa tus credenciales:

```bash
cp .env.example .env
```

```dotenv
# .env
HOME_ASSISTANT_URL=http://localhost:8123
HOME_ASSISTANT_TOKEN=tu_token_de_larga_duracion

LLM_PROVIDER=gemini        # gemini | openai
GEMINI_API_KEY=tu_api_key
OPENAI_API_KEY=tu_api_key

RTSP_CAMERA_URLS=rtsp://usuario:pass@192.168.1.50:554/onvif1
WAKE_WORD_MODEL=miles.onnx

DB_PATH=./data/miles.db
```

### 5. Inicializar la base de datos local

```bash
python scripts/init_db.py
```

### 6. Ejecutar Miles

```bash
python -m core.main
```

---

## 🔗 Ejemplo de Interacción entre Módulos

El siguiente fragmento ilustra, de forma simplificada, cómo el **orquestador** conecta el módulo de audio, el contexto almacenado y el motor LLM para generar una respuesta consciente del entorno:

```python
# core/orchestrator.py

from modules.audio import wake_word_listener, transcribe_audio, speak
from modules.presence import get_users_at_home
from modules.memory import ContextStore
from modules.llm import LLMEngine

context_store = ContextStore(db_path="./data/miles.db")
llm = LLMEngine(provider="gemini")

def main_loop():
    while True:
        # 1. Escucha pasiva y local de la wake word
        if wake_word_listener.detected("miles"):
            
            # 2. Transcripción de voz a texto (Whisper)
            user_command = transcribe_audio()

            # 3. Recuperar contexto actual del hogar
            present_users = get_users_at_home()
            house_state = context_store.get_current_state()

            # 4. Inyectar contexto en el prompt del LLM
            response = llm.generate_response(
                user_input=user_command,
                context={
                    "usuarios_presentes": present_users,
                    "estado_casa": house_state,
                }
            )

            # 5. Ejecutar acción en Home Assistant si aplica
            if response.action:
                context_store.execute_action(response.action)

            # 6. Responder por voz (TTS)
            speak(response.text)
```

> 💡 En este flujo, **la personalización no vive en el LLM**: vive en `ContextStore`, que actúa como la memoria de Miles y es responsable de decidir qué información del hogar es relevante inyectar en cada conversación.

---

## 🗺️ Roadmap

- [x] **Fase 1 — Infraestructura**
  Puesta en marcha del servidor Edge, Home Assistant vía Docker, red y cámaras con IP estática.

- [ ] **Fase 2 — Visión Artificial**
  Integración de streams RTSP, detección facial con `face_recognition` y almacenamiento de identidades en SQLite.

- [ ] **Fase 3 — Voz**
  Wake word offline con OpenWakeWord, pipeline STT (Whisper) y TTS (Edge-TTS / pyttsx3).

- [ ] **Fase 4 — Motor LLM y Memoria Contextual**
  Diseño del sistema de prompts dinámicos, integración con Gemini/OpenAI y motor de inyección de contexto.

- [ ] **Fase 5 — Interfaz Web**
  Dashboard de administración (estado del hogar, logs de conversación, gestión de perfiles y dispositivos).

- [ ] **Fase 6 — Refinamiento y Escalabilidad**
  Contenerización completa de todos los módulos, métricas, alertas y soporte multi-hogar.

---

## 🤝 Contribuciones

Este es un proyecto personal en desarrollo activo. Si tienes sugerencias, encuentras un bug o quieres proponer una mejora, siéntete libre de abrir un *issue* o un *pull request*.

## 📄 Licencia

Distribuido bajo la licencia MIT. Consulta el archivo `LICENSE` para más detalles.

---

<div align="center">
<sub>Construido con 🕷️ y mucho ☕ para un hogar más inteligente y privado.</sub>
</div>
