# Proyecto de Automatización VPN Full-Mesh (IPSec/GRE/OSPF)

Este repositorio contiene la **Fase 1** del proyecto de automatización de redes utilizando Ansible, orientado a configurar una topología Full-Mesh entre 3 routers MikroTik CHR. 

## Arquitectura de la Solución
El proyecto establece conectividad segura y enrutamiento dinámico automatizando la configuración de:
1. **Underlay (Físico)**: Direccionamiento IP en enlaces físicos punto a punto.
2. **Overlay (GRE)**: Túneles GRE para crear la topología lógica Full-Mesh.
3. **Seguridad (IPSec)**: Cifrado del tráfico GRE utilizando claves pre-compartidas de forma nativa en MikroTik.
4. **Enrutamiento Dinámico (OSPFv3/v7)**: Distribución de las redes Loopback a través de los túneles seguros.

## Estructura del Proyecto

```text
├── ansible.cfg              # Configuración base de Ansible (desactiva host_key_checking)
├── inventory/
│   └── hosts.yml            # Inventario con los 3 routers MikroTik (R1, R2, R3)
├── group_vars/
│   └── routers.yml          # Variables globales (IPSec Secret, OSPF Area)
├── host_vars/
│   ├── R1.yml               # Variables específicas de R1 (IPs de interfaces y GRE)
│   ├── R2.yml               # Variables específicas de R2 (IPs de interfaces y GRE)
│   └── R3.yml               # Variables específicas de R3 (IPs de interfaces y GRE)
├── roles/
│   ├── base_config/         # Rol: Loopbacks, Hostname e IPs físicas
│   ├── gre_tunnel/          # Rol: Interfaces GRE con IPSec secreto dinámico
│   └── ospf_routing/        # Rol: Instancias, Áreas y Templates OSPF
├── site.yml                 # Playbook principal que orquesta todos los roles
├── Dockerfile               # Entorno Docker con Python, Ansible y dependencias de red
└── docker-compose.yml       # Orquestación del contenedor
```

## Requisitos y Entorno
El código está diseñado para ejecutarse desde un entorno aislado utilizando **Docker**, evitando problemas de dependencias en el sistema host. Las librerías instaladas incluyen `netmiko`, `paramiko` y el collection `community.routeros`.

### Ejecución del Proyecto
1. **Construir el contenedor:**
   ```bash
   docker compose build
   ```
2. **Acceder a la consola del contenedor** (o usarlo como nodo en GNS3).
3. **Ejecutar el playbook principal:**
   ```bash
   ansible-playbook site.yml -e "ansible_password=TU_PASSWORD"
   ```

## Idempotencia y Manejo de Errores
El código utiliza módulos declarativos y comandos estructurados en bloques `block` / `rescue` dentro de cada rol. Esto asegura que si una configuración falla o el router es inalcanzable, Ansible capturará el error sin romper abruptamente la ejecución, reportando el fallo detallado. Las ejecuciones repetidas no duplicarán la configuración gracias a la naturaleza de las comprobaciones de estado.

## Escalabilidad y Reusabilidad
El diseño modular de este proyecto fue pensado específicamente para tener un **alto potencial de reusabilidad y escalabilidad**, facilitando enormemente su extensión a nuevos escenarios:

1. **Agnóstico de Código (Reusabilidad):** La lógica de configuración dentro de la carpeta `roles/` no tiene hardcodeada (fijada) ninguna dirección IP ni nombre de router. Esto significa que los roles pueden ser copiados y reutilizados en otros proyectos de la empresa sin cambiar ni una sola línea de código.
2. **Escalabilidad Horizontal Fácil:** Extender esta red a un nuevo dispositivo (por ejemplo, agregar una sucursal `R4` a la malla) toma menos de un minuto. No es necesario modificar los playbooks; basta con:
   - Agregar la IP de administración de `R4` en el archivo `inventory/hosts.yml`.
   - Crear un archivo de variables nuevo `host_vars/R4.yml` declarando sus túneles GRE y Loopbacks.
   - Ansible automáticamente recogerá el nuevo archivo, lo integrará al loop y armará el Full-Mesh incluyendo a `R4`.

## Integración y Despliegue Continuo (CI/CD) - Fase 2
En la **Fase 2** del proyecto, se implementaron flujos de trabajo automatizados usando **GitHub Actions** y un **Self-Hosted Runner** local. Esto permite aplicar principios de DevOps directamente sobre la topología virtualizada en GNS3.

### Arquitectura de los Pipelines
El repositorio cuenta con dos pipelines separados lógicamente (`.github/workflows/`):

1. **Early Testing (Pruebas Tempranas - Shift-Left)** (`early-testing.yml`):
   - **Gatillo (Trigger):** Se activa automáticamente al crear un *Pull Request* o subir cambios a ramas de desarrollo.
   - **Objetivo:** Detectar errores antes de que el código llegue a producción.
   - **Pruebas:** Ejecuta `yamllint` (formato), `ansible-lint` (mejores prácticas) y un `--syntax-check` estructurado.

2. **Continuous Deployment (Despliegue Continuo)** (`deploy.yml`):
   - **Gatillo (Trigger):** Se activa al hacer push en la rama `main` o al dispararlo manualmente desde la web (`workflow_dispatch`).
   - **Objetivo:** Aplicar las configuraciones a los routers en GNS3 usando Ansible.
   - **Lógica Condicional:** Si el mensaje del commit contiene la etiqueta `[skip deploy]`, el pipeline omite el despliegue a la red. Ideal para cambios de solo-lectura (como documentar el README).
   - **Notificaciones:** Al finalizar, envía un reporte (Éxito/Fallo) al correo electrónico del administrador vía SMTP.

### Manejo Seguro de Credenciales (Vault & Secrets)
Las contraseñas no se almacenan en texto plano en el repositorio. Los pipelines recuperan el secreto `ANSIBLE_VAULT_PASSWORD` desde **GitHub Secrets**, lo inyectan en un archivo temporal `.vault_pass` para descifrar el inventario, y destruyen el archivo inmediatamente después de usarlo.

### Instrucciones de Configuración y Ejecución
Para hacer funcionar el entorno CI/CD desde cero:

1. **Configurar el Self-Hosted Runner:**
   - En tu repositorio de GitHub ve a `Settings > Actions > Runners > New self-hosted runner`.
   - Copia los comandos en la terminal de la máquina física donde corre GNS3.
   - Mantén el runner escuchando ejecutando `./run.sh`.

2. **Aprovisionar los Secretos:**
   - En `Settings > Secrets and variables > Actions`, añade los siguientes secretos:
     - `ANSIBLE_VAULT_PASSWORD`: Tu contraseña de Ansible Vault.
     - `MAIL_TO`, `MAIL_USERNAME`, `MAIL_PASSWORD`: Credenciales para el envío de notificaciones por correo (SMTP).

3. **Verificación y Depuración de Logs:**
   - Para revisar ejecuciones, ve a la pestaña **Actions** en GitHub.
   - Gracias a las variables `ANSIBLE_STDOUT_CALLBACK: yaml` y colores forzados, los logs de Ansible se imprimen de forma altamente estructurada, facilitando una **depuración efectiva** si la sintaxis o el despliegue llegaran a fallar.
