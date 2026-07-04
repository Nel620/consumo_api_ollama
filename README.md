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

## 🧩 Lecciones aprendidas

- **Separación de responsabilidades:** el servidor solo infiere, el cliente local es el que arma el prompt y mide resultados.
- **Resiliencia de infraestructura:** el error 504 inicial se resolvió ajustando los timeouts de Nginx, en vez de evitar el problema.
- **Optimización de costos/recursos:** comparar modelos (Phi3 vs Llama3) según latencia real, en vez de asumir cuál es mejor.

---

## ⚠️ Nota sobre seguridad

Este proyecto expone Ollama en el puerto 80 sin autenticación, solo detrás de un subdominio. Aceptable para pruebas personales, pero no recomendado para producción sin agregar al menos autenticación básica (HTTP Basic Auth) o una API key en el proxy Nginx.
