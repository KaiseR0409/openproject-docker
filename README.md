# OpenProject Docker

OpenProject 17.7 en un solo contenedor Docker (all-in-one). Ideal para red local o produccion con Nginx + SSL.

## Inicio rapido

```bash
# Clonar el repo
git clone https://github.com/TU_USUARIO/openproject-docker.git
cd openproject-docker

# Configurar
cp .env.example .env
nano .env   # editar IP, secretos, etc.

# Levantar
sudo docker compose up -d

# Acceder
# http://TU_IP:8080
# Login: admin / admin
```

## Requisitos

- [Docker](https://docs.docker.com/get-docker/) >= 20.10
- [Docker Compose](https://docs.docker.com/compose/install/) >= 2.0
- 2GB de RAM minimo

## Archivos

| Archivo | Descripcion |
|---------|-------------|
| `docker-compose.yml` | Contenedor all-in-one (app + PostgreSQL) |
| `.env.example` | Plantilla de variables de entorno |
| `.env` | Tu configuracion real (no se sube a git) |

## Variables de entorno

| Variable | Descripcion | Default |
|----------|-------------|---------|
| `OPENPROJECT_HOST__NAME` | IP y puerto de acceso | `192.168.0.59:8080` |
| `PORT` | Puerto del host | `8080` |
| `OPENPROJECT_HTTPS` | Habilitar HTTPS | `false` |
| `SECRET_KEY_BASE` | Secreto para sesiones | `changeme` |
| `OPENPROJECT_DEFAULT__LANGUAGE` | Idioma por defecto | `es` |

---

## Uso basico

### Gestion del servicio

```bash
sudo docker compose up -d        # Levantar
sudo docker compose ps           # Ver estado
sudo docker compose logs -f      # Ver logs
sudo docker compose down         # Detener
sudo docker compose down -v      # Detener y borrar datos
sudo docker compose restart      # Reiniciar
```

### Acceso a consolas

```bash
# Consola Rails
sudo docker compose exec openproject bundle exec rails console

# PostgreSQL
sudo docker compose exec openproject psql -U openproject -d openproject
```

### Backup

```bash
# Base de datos
sudo docker compose exec openproject psql -U openproject -d openproject > backup_$(date +%Y%m%d).sql

# Archivos adjuntos
sudo tar -czf assets_$(date +%Y%m%d).tar.gz /var/lib/openproject/assets
```

---

## Gestion de usuarios

### Desde la web (recomendado)

1. Ir a **Administracion > Usuarios > + Usuario**
2. Rellenar: Login, nombre, apellido, email, idioma
3. Guardar y asignar contrasena manualmente

### Desde consola Rails

```bash
sudo docker compose exec openproject bundle exec rails console
```

```ruby
# Crear usuario
user = User.new(
  login: "nombre_usuario",
  firstname: "Nombre",
  lastname: "Apellido",
  mail: "correo@ejemplo.com",
  language: "es"
)
user.password = "contrasena_segura"
user.password_confirmation = "contrasena_segura"
user.status = 1  # 1 = activo
user.save!

# Listar usuarios
User.all.pluck(:id, :login, :firstname, :lastname, :mail, :status)

# Cambiar contrasena
u = User.find_by(login: "nombre_usuario")
u.password = "nueva_contrasena"
u.password_confirmation = "nueva_contrasena"
u.save!

# Deshabilitar usuario
u = User.find_by(login: "nombre_usuario")
u.status = 3  # 3 = bloqueado
u.save!

# Crear usuario y asignar a proyecto
user.save!
project = Project.find_by(identifier: "mi-proyecto")
role = Role.find_by(name: "Miembro")
Member.create!(user: user, project: project, roles: [role])
```

**Estados de usuario:**

| Valor | Estado |
|-------|--------|
| `1` | Activo |
| `2` | Invitado |
| `3` | Bloqueado |
| `4` | Registrado |

---

## Gestion de proyectos

```ruby
# Crear proyecto
project = Project.new(
  name: "Mi Proyecto",
  identifier: "mi-proyecto",
  description: "Descripcion del proyecto",
  is_public: false
)
project.save!

# Listar proyectos
Project.all.pluck(:id, :name, :identifier, :is_public)

# Crear sprint
Version.create!(project: project, name: "Sprint 1", status: "open")
```

---

## Gestion de work packages

```ruby
# Crear tipo
Type.create!(name: "Tarea", position: 1, color: "#0000FF", is_default: true)

# Crear estados
Status.create!(name: "En progreso", is_closed: false)
Status.create!(name: "Completado", is_closed: true)

# Crear work package
project = Project.find_by(identifier: "mi-proyecto")
wp = WorkPackage.new(
  project: project,
  subject: "Tarea de ejemplo",
  description: "Descripcion de la tarea",
  type: Type.find_by(name: "Tarea"),
  status: Status.find_by(name: "Nueva")
)
wp.save!
```

---

## Produccion: DNS + Nginx + SSL

### Arquitectura

```
Internet/Red -> Puerto 80/443 -> Nginx (SSL) -> 127.0.0.1:8080 -> OpenProject
```

### 1. Configurar DNS

**Opcion A - DNS local** (sin comprar dominio):

En cada PC client, editar el archivo de hosts:
- Windows: `C:\Windows\System32\drivers\etc\hosts`
- Linux/Mac: `/etc/hosts`

```
192.168.0.59    openproject.local
```

**Opcion B - DNS publico** (con dominio real):

En tu proveedor de DNS (Cloudflare, Namecheap, etc.), crear un registro A:
```
Nombre:  project
Valor:   IP publica de tu servidor
TTL:     Automatico
```

### 2. Instalar Nginx

```bash
sudo apt update && sudo apt install -y nginx
```

Crear configuracion `/etc/nginx/sites-available/openproject`:

```nginx
server {
    listen 80;
    server_name project.tuempresa.com;

    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_request_buffering off;

        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
    }
}
```

Activar y recargar:

```bash
sudo ln -s /etc/nginx/sites-available/openproject /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

### 3. Configurar SSL con Let's Encrypt

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d project.tuempresa.com

# Verificar renovacion automatica
sudo certbot renew --dry-run
```

### 4. Docker Compose para produccion

En produccion, NO expongas el puerto 8080 al mundo. Cambia el `docker-compose.yml`:

```yaml
services:
  openproject:
    image: openproject/openproject:17
    restart: always
    ports:
      - "127.0.0.1:8080:80"  # Solo accesible desde localhost
    environment:
      OPENPROJECT_HTTPS: "true"
      OPENPROJECT_HOST__NAME: "project.tuempresa.com"
      SECRET_KEY_BASE: "TU_SECRETO_AQUI"
      OPENPROJECT_DEFAULT__LANGUAGE: "es"
    volumes:
      - pgdata:/var/openproject/pgdata
      - opdata:/var/openproject/assets

volumes:
  pgdata:
  opdata:
```

### 5. Firewall

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow OpenSSH
sudo ufw status verbose
```

### 6. Router (acceso desde internet)

En tu router, crear Port Forwarding:
- Puerto externo `80` -> `192.168.0.59:80` TCP
- Puerto externo `443` -> `192.168.0.59:443` TCP

Si tu IP publica no es fija, usa [DuckDNS](https://duckdns.org) (gratis).

### Script de instalacion completa

```bash
#!/bin/bash
# install-production.sh - Ejecutar como root

set -e

DOMAIN="project.tuempresa.com"
EMAIL="admin@tuempresa.com"

apt update && apt install -y nginx certbot python3-certbot-nginx

cat > /etc/nginx/sites-available/openproject <<EOF
server {
    listen 80;
    server_name ${DOMAIN};
    client_max_body_size 100M;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
    }
}
EOF

ln -sf /etc/nginx/sites-available/openproject /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx

certbot --nginx -d ${DOMAIN} --non-interactive --agree-tos -m ${EMAIL}

ufw allow 80/tcp
ufw allow 443/tcp
ufw allow OpenSSH
ufw --force enable

echo "Listo: https://${DOMAIN}"
```

---

## Troubleshooting

| Problema | Solucion |
|----------|----------|
| No se accede desde otra maquina | `sudo ufw allow 8080/tcp`, verificar que `OPENPROJECT_HOST__NAME` tiene la IP correcta |
| Error `invalid host_name` | En `.env`: `OPENPROJECT_HOST__NAME=IP:PUERTO`, reiniciar compose |
| Contenedor no inicia (exit 127) | `sudo docker compose down -v` y volver a levantar |
| Lentitud | Verificar RAM con `sudo docker stats`, OpenProject 17 necesita 2GB minimo |

---

## Comandos utiles

```bash
# Verificar que Docker corre
sudo systemctl status docker

# Espacio en disco
sudo docker system df

# Limpiar imagenes
sudo docker image prune -f

# Verificar conectividad
ping TU_IP
curl -I http://TU_IP:8080

# Ver puertos en escucha
ss -tlnp | grep 8080

# Logs de Nginx
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# Verificar certificado
sudo certbot certificates
```

---

## Licencia

MIT
