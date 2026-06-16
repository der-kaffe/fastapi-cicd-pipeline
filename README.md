# cicd

API REST con un pipeline de integración y despliegue continuo completo. Cada push a `main` ejecuta las pruebas, construye la imagen, la escanea, la publica en Docker Hub y despliega en el servidor — sin intervención manual.

---

## Requisitos

### Máquina host

Probado en Debian 13 (trixie) con zsh, aunque debería funcionar en cualquier distro Linux moderna.

| Herramienta | Versión usada |
|---|---|
| Terraform | 1.15.3 |
| Ansible | core 2.19.4 |
| Docker | 29.5.3 |
| libvirt / KVM | incluido en Debian |
| virsh | incluido en libvirt |
| qemu-system-x86_64 | incluido en qemu |
| Python | 3.13 |

Instalar dependencias en Debian:

```bash
sudo apt install -y terraform ansible docker.io libvirt-daemon-system \
  libvirt-clients qemu-system-x86_64 virtinst python3 python3-pip
```

Instalar la colección de Ansible para Docker:

```bash
ansible-galaxy collection install community.docker
```

### Imagen de VM

Descargar la imagen cloud de Debian 12 y colocarla en `~/vmstore/images/`:

```bash
mkdir -p ~/vmstore/images
wget -P ~/vmstore/images \
  https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2
```

### Cuentas externas

- Cuenta en [Docker Hub](https://hub.docker.com) con un Access Token (Read & Write)
- Repositorio en GitHub con dos secrets configurados:
  - `DOCKERHUB_USERNAME`
  - `DOCKERHUB_TOKEN`

---

## Estructura

```
.
├── .github/
│   └── workflows/
│       └── ci.yml              # Definición completa del pipeline
├── app/
│   └── main.py                 # Aplicación FastAPI
├── tests/
│   └── test_main.py            # Pruebas con Pytest
├── infra/
│   ├── terraform/              # Aprovisionamiento de la VM
│   │   ├── main.tf
│   │   ├── provider.tf
│   │   ├── variables.tf
│   │   ├── terraform.tf
│   │   ├── config/
│   │   │   ├── cloud_init.cfg
│   │   │   └── network_config.cfg
│   │   ├── init.sh             # Levanta la VM y configura SSH
│   │   └── delete.sh           # Destruye la VM y limpia todo
│   └── ansible/                # Instalación de Docker y despliegue
│       ├── deploy.yml
│       ├── inventory.ini
│       └── ansible.cfg
├── actions-runner/             # Runner de GitHub Actions (no se sube al repo)
├── Dockerfile
├── requirements.txt
└── requirements-dev.txt
```

---

## Puesta en marcha

### 1. Clonar el repositorio

```bash
git clone https://github.com/der-kaffe/cicd.git
cd cicd
```

### 2. Entorno virtual de Python

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
```

### 3. Levantar la VM

```bash
cd infra/terraform
./init.sh
```

Este script inicializa Terraform, aprovisiona la VM con Debian 12 via KVM, espera a que arranque y verifica la conexión SSH. La llave privada se genera automáticamente en `infra/terraform/id_ed25519_debian`.

### 4. Instalar el runner de GitHub Actions

Ir a `Settings → Actions → Runners → New self-hosted runner` en el repositorio de GitHub, seleccionar Linux / x64 y seguir los comandos que aparecen ahí. Dejar el runner corriendo en una terminal aparte.

### 5. Configurar los secrets en GitHub

Ir a `Settings → Secrets and variables → Actions` y crear:

| Secret | Valor |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | Access Token de Docker Hub |

### 6. Primer deploy

```bash
git commit --allow-empty -m "ci: first deploy"
git push origin main
```

El pipeline completo debería terminar en verde en la pestaña Actions del repositorio.

---

## Pipeline

```
git push --> [Lint] --> [Tests] --> [Build + Scan] --> [Push Docker Hub] --> [Deploy]
```

- **Lint**: Ruff revisa el código
- **Tests**: Pytest corre la suite de pruebas
- **Build + Scan**: construye la imagen Docker y Trivy escanea vulnerabilidades críticas con parche disponible — si encuentra alguna, el pipeline falla
- **Push**: publica la imagen en Docker Hub con el SHA del commit y el tag `latest`
- **Deploy**: el runner local ejecuta el playbook de Ansible que instala Docker en la VM y despliega el contenedor

---

## Infraestructura

Levantar la VM desde cero:

```bash
cd infra/terraform
./init.sh
```

Destruir todo limpiamente:

```bash
cd infra/terraform
./delete.sh
```

El playbook de Ansible es idempotente: correrlo varias veces sobre una VM ya configurada no genera cambios no deseados.

---

## Ajustar variables

Si se quiere cambiar la IP, hostname, recursos de la VM u otros parámetros, editar `infra/terraform/variables.tf`. Si se cambia la IP, actualizar también `infra/ansible/inventory.ini`.

---

## API

| Endpoint | Descripción |
|---|---|
| `GET /` | Estado del servicio |
| `GET /health` | Health check con uptime y versión |

```bash
curl http://192.168.122.101:8000/health
# {"status":"healthy","uptime_seconds":42.1,"version":"1.0.0"}
```
