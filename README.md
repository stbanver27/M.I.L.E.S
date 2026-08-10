# Miles

Asistente de hogar inteligente construido desde cero, sin depender de Alexa, Google Home ni nada por el estilo. Corre en un servidor propio dentro de casa (un mini PC reacondicionado) y combina automatización, visión artificial y un LLM para tener conversaciones que realmente tienen en cuenta lo que está pasando en la casa.

El nombre es un guiño a Miles Morales — control de la "red" del hogar, en más de un sentido.

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Middleware-41BDF5?logo=home-assistant&logoColor=white)](https://www.home-assistant.io/)
[![Docker](https://img.shields.io/badge/Docker-Contenedores-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![OpenCV](https://img.shields.io/badge/OpenCV-Visi%C3%B3n-5C3EE8?logo=opencv&logoColor=white)](https://opencv.org/)
[![SQLite](https://img.shields.io/badge/SQLite-BD-003B57?logo=sqlite&logoColor=white)](https://www.sqlite.org/)
[![Whisper](https://img.shields.io/badge/Whisper-STT-000000?logo=openai&logoColor=white)](https://github.com/openai/whisper)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-Server-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)

---

## Por qué existe este proyecto

Los asistentes comerciales funcionan bien, pero todo pasa por la nube: el micrófono está siempre mandando audio afuera, y no hay forma de saber realmente qué hacen con eso. Quería algo que procesara lo sensible —la palabra de activación, el reconocimiento facial, saber quién está en casa— de forma local, en mi propio hardware, y que solo dependiera de una API externa para la parte de razonamiento conversacional (que es donde de verdad conviene usar un LLM grande).

La otra motivación era la personalización. No quiero un asistente genérico que responda lo mismo sin importar quién le hable. Miles guarda contexto — quién llegó, qué dispositivos están prendidos, qué pasó hace un rato — y se lo inyecta al LLM en cada conversación, así las respuestas tienen sentido con lo que está pasando realmente en la casa.

Todo está armado en módulos separados (audio, visión, presencia, lógica conversacional) en vez de un solo script gigante. Cada uno se puede tocar, probar y reemplazar sin romper el resto.

## Arquitectura

Hay un orquestador en Python que conecta todo. Home Assistant se encarga de hablar con los dispositivos físicos (así no tengo que lidiar yo mismo con cada protocolo distinto), y SQLite guarda el estado y el historial que después se usa como contexto para el LLM.

```
                                ┌───────────────────────────┐
                                │      Usuario (voz)        │
                                └─────────────┬──────────────┘
                                              │
                     ┌────────────────────────▼────────────────────────┐
                     │               Módulo de audio                   │
                     │  OpenWakeWord → Whisper (STT) → Edge-TTS (TTS)  │
                     └────────────────────────┬────────────────────────┘
                                              │ texto / intención
                                              ▼
     ┌───────────────────┐        ┌───────────────────────┐        ┌───────────────────┐
     │  Módulo de visión │◄──────►│      Orquestador        │◄──────►│  Módulo de red     │
     │  OpenCV + RTSP     │        │      (core Python)     │        │  Escaneo Wi-Fi/MAC │
     │  face_recognition  │        │                          │        │  Presencia         │
     └─────────┬───────────┘        └───────────┬────────────┘        └─────────┬───────────┘
               │                                  │                                │
               ▼                                  ▼                                ▼
     ┌────────────────────────────────────────────────────────────────────────────────┐
     │                        SQLite — memoria de Miles                                │
     │        perfiles · historial de eventos · estado de dispositivos · logs         │
     └───────────────────────────────────┬────────────────────────────────────────────┘
                                          │ contexto para el prompt
                                          ▼
                     ┌───────────────────────────────────────┐
                     │            LLM (Gemini / OpenAI)       │
                     └────────────────────┬────────────────────┘
                                          │ acción / respuesta
                                          ▼
                     ┌───────────────────────────────────────┐
                     │        Home Assistant (Docker)         │
                     └────────────────────┬────────────────────┘
                                          │
                                          ▼
                        TVs · luces · enchufes · cámaras
```

Un par de decisiones de diseño que valen la pena mencionar:

- Los módulos no se hablan directamente entre sí — todo pasa por el orquestador y la base de datos compartida. Menos acoplamiento, más fácil de debuggear.
- Home Assistant es la capa que absorbe la variedad de protocolos (ONVIF, MQTT, Zigbee...). Si mañana cambio de marca de cámaras o agrego un enchufe nuevo, la lógica de Miles no se entera.
- Todo lo que se puede procesar local, se procesa local. La nube entra recién al final, para la parte conversacional.

## Lo que necesitás para correrlo

**Hardware**

- Un servidor casero: en mi caso un mini PC x86 con i5, 8GB de RAM y SSD. No hace falta mucho más.
- Micrófono de campo lejano (uso un ReSpeaker USB) y parlantes.
- Cámaras IP compatibles con ONVIF/RTSP — las mías son Yoosee, con IP fija.
- Conexión Ethernet para el servidor (Wi-Fi da problemas con el streaming constante de las cámaras).

**Software**

- Ubuntu 22.04 o superior
- Python 3.10+
- Docker y Docker Compose
- `ffmpeg`
- API key de Gemini y/o OpenAI
- Acceso al router, para configurar IPs estáticas y poder escanear la red

## Instalación

Clonar el repo:

```bash
git clone https://github.com/tu-usuario/miles.git
cd miles
```

Levantar Home Assistant con Docker:

```bash
docker compose -f infra/homeassistant/docker-compose.yml up -d
```

Entrá a `http://localhost:8123`, hacé la configuración inicial y generá un Long-Lived Access Token — Miles lo necesita para hablar con la API de Home Assistant.

Crear el entorno virtual e instalar dependencias:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Copiar el archivo de variables de entorno y completarlo con tus datos:

```bash
cp .env.example .env
```

```dotenv
HOME_ASSISTANT_URL=http://localhost:8123
HOME_ASSISTANT_TOKEN=tu_token_de_larga_duracion

LLM_PROVIDER=gemini        # gemini | openai
GEMINI_API_KEY=tu_api_key
OPENAI_API_KEY=tu_api_key

RTSP_CAMERA_URLS=rtsp://usuario:pass@192.168.1.50:554/onvif1
WAKE_WORD_MODEL=miles.onnx

DB_PATH=./data/miles.db
```

Inicializar la base de datos:

```bash
python scripts/init_db.py
```

Y correr Miles:

```bash
python -m core.main
```

## Cómo se conectan los módulos, en la práctica

Este es más o menos el flujo real cuando alguien le habla a Miles: se despierta con la wake word, transcribe lo que dijiste, junta el contexto que tiene guardado (quién está en casa, qué estado tiene la casa) y se lo pasa al LLM junto con tu mensaje.

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
        if wake_word_listener.detected("miles"):
            user_command = transcribe_audio()

            present_users = get_users_at_home()
            house_state = context_store.get_current_state()

            response = llm.generate_response(
                user_input=user_command,
                context={
                    "usuarios_presentes": present_users,
                    "estado_casa": house_state,
                }
            )

            if response.action:
                context_store.execute_action(response.action)

            speak(response.text)
```

La parte importante acá es que la personalización no la hace el LLM — la hace `ContextStore`, que decide qué información de la casa vale la pena mandarle en cada momento. El LLM solo redacta la respuesta con lo que le dieron.

## Roadmap

- [x] **Infraestructura** — servidor levantado, Home Assistant corriendo en Docker, red y cámaras configuradas con IP fija.
- [ ] **Visión artificial** — conectar los streams RTSP, reconocimiento facial con `face_recognition` y guardar identidades en SQLite.
- [ ] **Voz** — wake word offline con OpenWakeWord, y el pipeline de STT/TTS con Whisper y Edge-TTS.
- [ ] **LLM y memoria contextual** — armar bien el sistema de prompts, conectar Gemini/OpenAI e inyectar el contexto de la casa.
- [ ] **Interfaz web** — un panel simple para ver el estado de la casa, revisar el historial de conversaciones y administrar perfiles.
- [ ] **Pulido y escalabilidad** — contenerizar todo, agregar métricas/alertas, y ver si tiene sentido soportar más de un hogar.

## Contribuciones

Es un proyecto personal que voy armando en mi tiempo libre. Si tenés una idea, encontrás un bug o querés proponer algo, abrí un issue o un PR.

## Licencia

MIT — ver el archivo `LICENSE`.
