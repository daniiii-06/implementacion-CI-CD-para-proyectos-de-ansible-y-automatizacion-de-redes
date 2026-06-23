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
