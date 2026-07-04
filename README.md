# 🧠 IA Local Gratuita: Servidor de Inferencia con Ollama (llama3 vs phi3)

Proyecto de laboratorio personal para montar un motor de inferencia de LLMs **propio y gratuito** en la nube, exponerlo de forma segura mediante un proxy inverso, y compararlo con distintos modelos sin depender de APIs de pago (OpenAI, Anthropic, etc.).

> ⚠️ Proyecto de práctica/aprendizaje: el objetivo fue entender de punta a punta cómo desplegar y consumir un LLM propio, no un sistema en producción.

---

## 🎯 Objetivo

Construir una arquitectura cliente-servidor donde:
- El **cómputo pesado (LLM)** vive en un servidor remoto barato (Droplet de DigitalOcean).
- El **cliente** (Python/Jupyter) le hace peticiones por HTTP y mide tiempos de respuesta.
- Todo esto sin pagar por tokens de ningún proveedor comercial.

---

## 🏗️ Arquitectura del sistema

```
┌─────────────────────────┐          HTTP (puerto 80)          ┌──────────────────────────────┐
│   ENTORNO LOCAL          │ ─────────────────────────────────▶ │   SERVIDOR (DigitalOcean)     │
│   Python / Jupyter       │                                     │   Ubuntu + Nginx + Ollama     │
│                          │ ◀───────────────────────────────── │                                │
│  - ollama-client (envía  │          Respuesta del modelo       │  Nginx (proxy inverso)         │
│    prompts y mide tiempo)│                                     │   ↓ proxy_pass                 │
└─────────────────────────┘                                     │  Ollama (puerto 11434)          │
                                                                  │   ↓                             │
                                                                  │  Modelo LLM (llama3 / phi3)     │
                                                                  └──────────────────────────────┘
```

**Componentes:**

| Capa | Tecnología | Función |
|---|---|---|
| Infraestructura | DigitalOcean Droplet (2 vCPU, 8GB RAM) | Servidor donde corre todo |
| Motor de inferencia | [Ollama](https://ollama.com) (instalación nativa) | Ejecuta los modelos LLM en CPU |
| Proxy inverso | Nginx | Expone Ollama de forma controlada y estable |
| DNS | Namecheap (subdominio `api.entornoandres.me`) | Enmascara la IP del servidor |
| Cliente | Python + librería `ollama` | Envía prompts y mide tiempos de respuesta |

El servidor **no tiene Python, ni Docker, ni código de negocio**. Su única función es recibir peticiones HTTP y devolver inferencias. Todo el consumo se hace desde el entorno local.

---

## ⚙️ Instalación y despliegue del servidor

### 1. Instalación de Ollama

```bash
# Instalar Ollama nativamente
curl -fsSL https://ollama.com/install.sh | sh

# Configurar el servicio para aceptar conexiones externas
sudo sed -i '/\[Service\]/a Environment="OLLAMA_HOST=0.0.0.0"' /etc/systemd/system/ollama.service
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 2. Configuración del proxy inverso (Nginx)

```bash
sudo apt update && sudo apt install nginx -y

sudo bash -c 'cat > /etc/nginx/sites-available/default <<EOF
server {
    listen 80;
    server_name api.entornoandres.me;

    location / {
        proxy_pass http://127.0.0.1:11434;
        proxy_read_timeout 300s;
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
    }
}
EOF'

sudo systemctl restart nginx
```

> 💡 **¿Por qué 300s de timeout?** Los modelos LLM corriendo en CPU (sin GPU) pueden tardar decenas de segundos en responder. Sin este ajuste, Nginx corta la conexión con un error 504 antes de que el modelo termine de generar la respuesta.

### 3. Descargar los modelos en el servidor

```bash
ollama pull llama3
ollama pull phi3
```

---

## 🚀 Comparación de modelos: Llama3 vs Phi3

Ambos modelos fueron probados con el mismo prompt (cálculo de crecimiento de ventas Q1→Q2) desde el cliente local, midiendo el tiempo de respuesta con `time.time()`.

| Modelo | Tipo | Resultado | Latencia observada |
|---|---|---|---|
| **Phi3** | SLM (Small Language Model) | 40% (correcto) | ~34 segundos |
| **Llama3 (8B)** | LLM (Large Language Model) | 40% (correcto) | ~58 segundos |

Ambos llegaron al resultado correcto, pero Phi3 fue notablemente más rápido — la ventaja esperada de un modelo más chico corriendo en CPU sin GPU dedicada.

Notebooks de la comparación en [`notebooks/`](./notebooks):
- [`llama3.ipynb`](./notebooks/llama3.ipynb)
- [`phi3.ipynb`](./notebooks/phi3.ipynb)

---

## 📁 Estructura del repositorio

```
proyecto-ia-local/
├── README.md
└── notebooks/
    ├── llama3.ipynb   # Prueba de inferencia con Llama3
    └── phi3.ipynb     # Prueba de inferencia con Phi3
```

---

## ⏱️ Diagnóstico de Latencia: ¿Por qué tarda 50-60 segundos?

El sistema no presenta errores (bugs), sino una limitación física por arquitectura CPU-bound. Tras realizar las pruebas de conectividad y monitoreo (htop), se descartaron fallos de configuración:

- Red: Conectividad estable entre el cliente local y el Proxy Nginx.
- RAM: Carga óptima del modelo en memoria física (Swap: 0), sin I/O Wait.
- Cómputo: El tiempo de respuesta es la tasa de procesamiento secuencial de un CPU de 2 vCPUs sin aceleración por GPU.

**Conclusión:** El despliegue es funcional y estable. La latencia observada de ~50-60s es el comportamiento estándar de hardware para inferencia de LLMs sin GPU dedicada en CPUs de 2 vCPUs.

---

### 💡 Hoja de Ruta para Optimización (Escalabilidad)

Si el requerimiento de negocio exigiera tiempos de respuesta en milisegundos, la ruta técnica es:

1. **Migración a Aceleración por Hardware:** Sustituir la instancia estándar por un **GPU Droplet (NVIDIA CUDA)**. Esto reduciría el tiempo de inferencia de segundos a milisegundos.
2. **Implementación de Streaming:** en vez de esperar la respuesta completa, usar el modo streaming de la librería de Ollama. El script empieza a recibir palabras a medida que se generan, dando sensación de respuesta inmediata..
3. **Optimización de Latencia de Red:** Despliegue en regiones cloud próximas al usuario final (ej. `nyc` a `bogotá` tiene latencia inherente; un nodo en `sa-east-1` en Brasil o servicios locales reduciría el RTT).

---

## 🧩 Lecciones aprendidas

- **Arquitectura desacoplada:** separar el motor de inferencia (servidor) del cliente que arma los prompts permite escalar o reemplazar cualquiera de las dos partes sin tocar la otra — por ejemplo, cambiar de Droplet CPU a uno con GPU no obligaría a reescribir el código del cliente.
- **Diagnóstico antes que optimización:** el error 504 no se resolvió a ciegas subiendo el timeout por probar; primero se descartó red (curl local), memoria (swap en 0) y solo después se identificó el cómputo en CPU como el verdadero cuello de botella. Ese orden — red, memoria, cómputo — es el mismo que se usaría para diagnosticar cualquier sistema lento en producción.
- **El hardware importa más que el modelo:** comparar Phi3 vs Llama3 en el mismo servidor mostró que la diferencia de latencia entre un SLM y un LLM (34s vs 58s) es real, pero ambas cifras siguen siendo altas para uso interactivo — la limitación de fondo es la ausencia de GPU, no la elección del modelo.
- **Costo cero tiene un precio:** montar el modelo propio evita pagar por token, pero cambia ese costo por latencia y por el trabajo de mantener la infraestructura (proxy, timeouts, systemd). Es un trade-off consciente, no una solución gratis en el sentido estricto.

---

## 📸 Evidencia del Despliegue

Servidor activo (DigitalOcean Droplet):
<img width="1016" height="494" alt="servidor" src="https://github.com/user-attachments/assets/f6641070-d4c5-4419-a136-60a78e5c0f16" />

Instalación de Ollama:
<img width="1069" height="469" alt="ollama y llama 3" src="https://github.com/user-attachments/assets/069563da-ac93-4fd8-b4ca-aab2f41823ec" />

Servicio systemd corriendo:
<img width="1018" height="356" alt="Servicio systemd corriendo" src="https://github.com/user-attachments/assets/ba06e987-a028-4ab7-80f2-8a6cc3ab4aea" />

Configuración de Nginx: 
<img width="1017" height="322" alt="nginx" src="https://github.com/user-attachments/assets/40810f8d-365f-471a-9744-8e33994c2bde" />.

Prueba de conectividad:
<img width="1018" height="458" alt="curl" src="https://github.com/user-attachments/assets/8a4e57eb-011a-45bd-9b7f-ede285f65274" />

Diagnóstico de recursos: 
<img width="1020" height="678" alt="recursos" src="https://github.com/user-attachments/assets/6256036a-9ab4-4a5e-bbfd-52a6256f3239" />

Ejecución del notebook:
<img width="1282" height="617" alt="Jupyter" src="https://github.com/user-attachments/assets/c8ec55db-2c8b-4c0e-8978-c30b71aca9d9" />

---

## ⚠️ Nota sobre seguridad

Este proyecto expone Ollama en el puerto 80 sin autenticación, solo detrás de un subdominio. Aceptable para pruebas personales, pero no recomendado para producción sin agregar al menos autenticación básica (HTTP Basic Auth) o una API key en el proxy Nginx.
