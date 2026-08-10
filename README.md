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

Los asistentes comerciales (Alexa, Google Home, Siri) funcionan bien, pero todo pasa por la nube: el micrófono está mandando audio afuera todo el tiempo y realmente no hay forma de saber qué hacen con eso. **Miles** nace de querer algo distinto: un asistente construido desde cero donde lo sensible —la activación por voz, el reconocimiento facial, saber quién está en casa— se procesa localmente, en mi propio servidor, y la nube entra recién al final, solo para la parte de razonamiento conversacional.

El nombre es un guiño a Miles Morales (Spider-Man): control de la "red" tecnológica del hogar, en más de un sentido.

La otra motivación es la personalización real. No quería un asistente genérico que responda lo mismo sin importar quién le hable. Miles guarda contexto —quién llegó, qué dispositivos están prendidos, qué pasó hace un rato— y se lo inyecta al LLM en cada conversación, para que las respuestas tengan sentido con lo que está pasando en la casa (ej. *"Hola Juan, vi que acabas de llegar. Tu TV está encendida"*).

Todo está armado en módulos separados en vez de un solo script gigante: audio, visión, presencia y lógica conversacional se pueden tocar, probar y reemplazar de forma independiente.

---

## 🏗️ Arquitectura de Software

Un orquestador central en Python conecta todo. Home Assistant se encarga de hablar con los dispositivos físicos (así no tengo que lidiar yo mismo con cada protocolo distinto), y SQLite guarda el estado y el historial que después se usa como contexto para el LLM.

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

Algunas decisiones que valen la pena explicar:

- **Sin acoplamiento directo entre módulos:** todo pasa por el orquestador y la base de datos compartida, nunca un módulo le habla a otro directamente. Es más fácil de debuggear y de reemplazar piezas sueltas.
- **Home Assistant como capa de abstracción:** absorbe la variedad de protocolos (ONVIF, MQTT, Zigbee...). Si mañana cambio de marca de cámaras o sumo un enchufe nuevo, la lógica de Miles ni se entera.
- **Edge-first:** todo lo que se puede procesar localmente (wake word, STT, visión) se procesa localmente. La nube solo entra para el razonamiento conversacional complejo.

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

Entrá a `http://localhost:8123`, completá la configuración inicial y generá un **Long-Lived Access Token** — Miles lo necesita para comunicarse con la API de Home Assistant.

### 3. Crear entorno virtual e instalar dependencias de Python

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 4. Configurar variables de entorno

Copiá el archivo de ejemplo y completalo con tus credenciales:

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

Así se ve, simplificado, el flujo real cuando alguien le habla a Miles: se despierta con la wake word, transcribe lo que dijiste, junta el contexto que tiene guardado (quién está en casa, en qué estado está la casa) y se lo pasa al LLM junto con tu mensaje.

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

> 💡 Lo importante acá es que la personalización no la hace el LLM — la hace `ContextStore`, que decide qué información de la casa vale la pena mandarle en cada momento. El LLM solo redacta la respuesta con lo que le dieron.

---

## 🗺️ Roadmap

- [x] **Fase 1 — Infraestructura**
  Servidor Edge levantado, Home Assistant corriendo en Docker, red y cámaras configuradas con IP estática.

- [ ] **Fase 2 — Visión Artificial**
  Conectar los streams RTSP, reconocimiento facial con `face_recognition` y guardar identidades en SQLite.

- [ ] **Fase 3 — Voz**
  Wake word offline con OpenWakeWord, y el pipeline de STT/TTS con Whisper y Edge-TTS / pyttsx3.

- [ ] **Fase 4 — Motor LLM y Memoria Contextual**
  Armar bien el sistema de prompts, conectar Gemini/OpenAI e inyectar el contexto de la casa.

- [ ] **Fase 5 — Interfaz Web**
  Un panel simple para ver el estado de la casa, revisar el historial de conversaciones y administrar perfiles y dispositivos.

- [ ] **Fase 6 — Refinamiento y Escalabilidad**
  Contenerizar todo por completo, sumar métricas y alertas, y ver si tiene sentido soportar más de un hogar.

---

## 🤝 Contribuciones

Este es un proyecto personal que voy armando en mi tiempo libre. Si tenés una sugerencia, encontrás un bug o querés proponer una mejora, abrí un *issue* o un *pull request*.

## 📄 Licencia

Distribuido bajo la licencia MIT. Mirá el archivo `LICENSE` para más detalles.

---

<div align="center">
<sub>Construido con 🕷️ y mucho ☕ para un hogar más inteligente y privado.</sub>
</div>
