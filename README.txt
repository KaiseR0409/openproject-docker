============================================================
  OPENPROJECT v17.7 - GUIA DE COMANDOS Y GESTION DE USUARIOS
  Red local: 192.168.0.59:8080
  Login admin: admin / admin
============================================================


------------------------------------------------------------
  1. COMANDOS DOCKER - GESTION DEL SERVICIO
------------------------------------------------------------

# Levantar el contenedor
sudo docker compose up -d

# Ver estado
sudo docker compose ps

# Ver logs en tiempo real
sudo docker compose logs -f

# Detener
sudo docker compose down

# Detener y eliminar volumenes (BORRA TODOS LOS DATOS)
sudo docker compose down -v

# Reiniciar
sudo docker compose restart

# Acceder a la consola Rails
sudo docker compose exec openproject bundle exec rails console

# Acceder a PostgreSQL
sudo docker compose exec openproject psql -U openproject -d openproject

# Verificar que responde
curl -I http://192.168.0.59:8080


------------------------------------------------------------
  2. GESTION DE USUARIOS - DESDE LA WEB (RECOMENDADO)
------------------------------------------------------------

Ruta: http://192.168.0.59:8080
Login admin: admin / admin

a) Crear usuario nuevo:
   - Ir a: Administracion > Usuarios > + Usuario
   - Rellenar: Login, nombre, apellido, email, idioma
   - Guardar y asignar contraseña manualmente

b) Editar usuario existente:
   - Administracion > Usuarios > clic en el usuario
   - Cambiar estado, contraseña, permisos

c) Asignar a un grupo:
   - Administracion > Grupos > crear o editar grupo
   - Agregar usuarios al grupo

d) Gestionar roles:
   - Administracion > Roles y permisos
   - Crear rol personalizado (ej: "Equipo", "Lector")
   - Asignar permisos: ver proyecto, editar work packages, etc.


------------------------------------------------------------
  3. GESTION DE USUARIOS - DESDE CONSOLA RAILS
------------------------------------------------------------

# Entrar a la consola Rails
sudo docker compose exec openproject bundle exec rails console

# --- CREAR USUARIO ---
user = User.new(
  login: "nombre_usuario",
  firstname: "Nombre",
  lastname: "Apellido",
  mail: "correo@ejemplo.com",
  language: "es"
)
user.password = "contraseña_segura"
user.password_confirmation = "contraseña_segura"
user.status = 1  # 1 = activo
user.save!

# --- LISTAR TODOS LOS USUARIOS ---
User.all.pluck(:id, :login, :firstname, :lastname, :mail, :status)

# --- BUSCAR USUARIO POR EMAIL ---
User.find_by(mail: "correo@ejemplo.com")

# --- CAMBIAR CONTRASEÑA DE UN USUARIO ---
u = User.find_by(login: "nombre_usuario")
u.password = "nueva_contraseña"
u.password_confirmation = "nueva_contraseña"
u.save!

# --- DESHABILITAR USUARIO (sin borrarlo) ---
u = User.find_by(login: "nombre_usuario")
u.status = 3  # 3 = bloqueado
u.save!

# --- HABILITAR USUARIO ---
u = User.find_by(login: "nombre_usuario")
u.status = 1  # 1 = activo
u.save!

# --- ELIMINAR USUARIO ---
u = User.find_by(login: "nombre_usuario")
u.destroy

# --- CREAR USUARIO Y ASIGNAR A PROYECTO ---
user = User.new(
  login: "nuevo_usuario",
  firstname: "Nombre",
  lastname: "Apellido",
  mail: "nuevo@ejemplo.com",
  language: "es"
)
user.password = "contraseña"
user.password_confirmation = "contraseña"
user.status = 1
user.save!

project = Project.find_by(identifier: "mi-proyecto")
role = Role.find_by(name: "Miembro")
Member.create!(user: user, project: project, roles: [role])


------------------------------------------------------------
  4. GESTION DE PROYECTOS - DESDE CONSOLA RAILS
------------------------------------------------------------

# --- CREAR PROYECTO ---
project = Project.new(
  name: "Mi Proyecto",
  identifier: "mi-proyecto",
  description: "Descripcion del proyecto",
  is_public: false
)
project.save!

# --- LISTAR PROYECTOS ---
Project.all.pluck(:id, :name, :identifier, :is_public)

# --- ELIMINAR PROYECTO ---
Project.find_by(identifier: "mi-proyecto").destroy

# --- CREAR CATEGORIA EN PROYECTO ---
project = Project.find_by(identifier: "mi-proyecto")
Category.create!(project: project, name: "Desarrollo")

# --- CREAR VERSION (SPRINT) EN PROYECTO ---
project = Project.find_by(identifier: "mi-proyecto")
Version.create!(project: project, name: "Sprint 1", status: "open")


------------------------------------------------------------
  5. WORK PACKAGES (TAREAS) - DESDE CONSOLA RAILS
------------------------------------------------------------

# --- CREAR TIPO DE WORK PACKAGE ---
Type.create!(
  name: "Tarea",
  position: 1,
  color: "#0000FF",
  is_default: true
)

# --- CREAR ESTADO ---
Status.create!(name: "En progreso", is_closed: false)
Status.create!(name: "Completado", is_closed: true)

# --- CREAR WORK PACKAGE ---
project = Project.find_by(identifier: "mi-proyecto")
wp = WorkPackage.new(
  project: project,
  subject: "Tarea de ejemplo",
  description: "Descripcion de la tarea",
  type: Type.find_by(name: "Tarea"),
  status: Status.find_by(name: "Nueva")
)
wp.save!

# --- LISTAR WORK PACKAGES DE UN PROYECTO ---
project = Project.find_by(identifier: "mi-proyecto")
project.work_packages.pluck(:id, :subject, :status_id)

# --- CAMBIAR ESTADO DE UN WORK PACKAGE ---
wp = WorkPackage.find(1)
wp.status = Status.find_by(name: "Completado")
wp.save!


------------------------------------------------------------
  6. COMANDOS UTILES DE POSTGRESQL
------------------------------------------------------------

# Entrar a PostgreSQL
sudo docker compose exec openproject psql -U openproject -d openproject

# Listar tablas
\dt

# Contar usuarios
SELECT COUNT(*) FROM users;

# Listar usuarios con su estado
SELECT id, login, firstname, lastname, mail, status FROM users;

# Estados de usuario:
#   1 = activo
#   2 = invitado
#   3 = bloqueado
#   4 = registrado

# Contar proyectos
SELECT COUNT(*) FROM projects;

# Listar proyectos
SELECT id, name, identifier, is_public FROM projects;

# Salir
\q


------------------------------------------------------------
  7. COMANDOS DE SISTEMA
------------------------------------------------------------

# Verificar que Docker esta corriendo
sudo systemctl status docker

# Reiniciar Docker si hay problemas
sudo systemctl restart docker

# Ver espacio en disco usado por Docker
sudo docker system df

# Limpiar imagenes no usadas
sudo docker image prune -f

# Verificar conectividad desde otra maquina
ping 192.168.0.59

# Probar acceso al servicio
curl http://192.168.0.59:8080

# Ver puertos en escucha
ss -tlnp | grep 8080


------------------------------------------------------------
  8. TROUBLESHOOTING
------------------------------------------------------------

PROBLEMA: No se puede acceder desde otra maquina
SOLUCION:
  - Verificar firewall: sudo ufw allow 8080/tcp
  - Verificar que el puerto esta bindeado a 0.0.0.0:
    ss -tlnp | grep 8080
  - Verificar que OPENPROJECT_HOST__NAME tiene la IP correcta

PROBLEMA: Error "invalid host_name configuration"
SOLUCION:
  - En .env: OPENPROJECT_HOST__NAME=192.168.0.59:8080
  - Reiniciar: sudo docker compose down && sudo docker compose up -d

PROBLEMA: Contenedor no inicia (exit code 127)
SOLUCION:
  - Limpiar volumenes viejos: sudo docker compose down -v
  - Volver a levantar: sudo docker compose up -d

PROBLEMA: Lentitud o timeouts
SOLUCION:
  - Verificar recursos: sudo docker stats
  - OpenProject 17 necesita al menos 2GB de RAM


------------------------------------------------------------
  9. ESTRUCTURA DE ARCHIVOS
------------------------------------------------------------

openproject-docker/
  docker-compose.yml    <-- Contenedor all-in-one (app + PostgreSQL)
  .env                  <-- Variables de entorno (IP, puertos, secretos)
  README.txt            <-- Este archivo

Datos persistentes (volumenes Docker):
  pgdata                <-- Base de datos PostgreSQL
  opdata                <-- Assets y archivos de OpenProject


============================================================
  10. PRODUCCION - DNS + NGINX + SSL
============================================================

------------------------------------------------------------
  A) CONFIGURAR UN NOMBRE DE DOMINIO (DNS)
------------------------------------------------------------

Opcion 1: DNS local (sin comprar dominio)
  Funciona solo desde tu red local.

  En cada PC client, editar el archivo de hosts:
    Windows: C:\Windows\System32\drivers\etc\hosts
    Linux/Mac: /etc/hosts

  Agregar esta linea:
    192.168.0.59    openproject.local

  Ahora accede a: http://openproject.local:8080


Opcion 2: DNS publico (con dominio real)
  Requisitos:
    - Un dominio comprado (ej: tuempresa.com)
    - Un servidor con IP publica fija (oDDNS)
    - Abrir puertos 80 y 443 en el router hacia tu servidor

  Pasos:
    1. En tu proveedor de DNS (Cloudflare, Namecheap, etc.),
       crear un registro A:
         Nombre:  project (o el subdominio que quieras)
         Valor:   IP publica de tu servidor
         TTL:     Automatico

    2. Esto te da: project.tuempresa.com

    3. Verificar que el DNS resuelve:
       nslookup project.tuempresa.com


------------------------------------------------------------
  B) INSTALAR NGINX COMO REVERSE PROXY
------------------------------------------------------------

# Instalar Nginx
sudo apt update && sudo apt install -y nginx

# Verificar que esta corriendo
sudo systemctl status nginx

# Crear configuracion para OpenProject
sudo nano /etc/nginx/sites-available/openproject

# Pegar esta configuracion (ajusta el server_name):

  server {
      listen 80;
      server_name project.tuempresa.com;  # <-- TU DOMINIO

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

# Activar la configuracion
sudo ln -s /etc/nginx/sites-available/openproject /etc/nginx/sites-enabled/

# Eliminar config default (opcional)
sudo rm /etc/nginx/sites-enabled/default

# Probar configuracion
sudo nginx -t

# Recargar Nginx
sudo systemctl reload nginx


------------------------------------------------------------
  C) CONFIGURAR SSL CON LET'S ENCRYPT (CERTIFICADO GRATIS)
------------------------------------------------------------

# Instalar Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtener certificado (Nginx configura automaticamente)
sudo certbot --nginx -d project.tuempresa.com

# Seguir las instrucciones en pantalla:
#   - Ingresar email
#   - Aceptar terminos
#   - Elegir redireccion automatico HTTP -> HTTPS

# Verificar renovacion automatica
sudo certbot renew --dry-run

# El certificado se renueva automaticamente cada 90 dias.
# Certbot crea un cron job o timer automaticamente.


------------------------------------------------------------
  D) ACTUALIZAR DOCKER COMPOSE PARA PRODUCCION
------------------------------------------------------------

Crear docker-compose.prod.yml al lado del docker-compose.yml:

  services:
    openproject:
      image: openproject/openproject:17
      restart: always
      environment:
        OPENPROJECT_HTTPS: "true"
        OPENPROJECT_HOST__NAME: "project.tuempresa.com"
        SECRET_KEY_BASE: "TU_SECRETO_AQUI"
        OPENPROJECT_DEFAULT__LANGUAGE: "es"
      volumes:
        - /var/lib/openproject/pgdata:/var/openproject/pgdata
        - /var/lib/openproject/assets:/var/openproject/assets

NOTA: En produccion NO se expone el puerto 8080 al mundo.
Nginx se encarga de escuchar en puertos 80/443 y redirigir
a OpenProject internamente.

El docker-compose.yml original queda como esta:
  services:
    openproject:
      image: openproject/openproject:17
      restart: always
      ports:
        - "127.0.0.1:8080:80"  # <-- Solo accesible desde localhost
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

Clave: el puerto se bindea a 127.0.0.1:8080 para que solo
Nginx pueda acceder a el, no el mundo exterior.


------------------------------------------------------------
  E) ABRIR PUERTOS EN EL FIREWALL
------------------------------------------------------------

# Solo puertos HTTP/HTTPS (Nginx)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow OpenSSH           # Para no bloquear SSH

# NO abrir el puerto 8080 en produccion
# Nginx se encarga del trafico externo

# Verificar reglas
sudo ufw status verbose


------------------------------------------------------------
  F) CONFIGURAR EL ROUTER (ACCESO DESDE FUERA DE LA RED)
------------------------------------------------------------

Si quieres acceder desde internet (fuera de tu red local):

  1. En tu router, buscar "Port Forwarding" o "Virtual Server"
  2. Crear regla:
       Puerto externo: 80
       Puerto interno: 80
       IP destino:     192.168.0.59
       Protocolo:      TCP

  3. Crear regla:
       Puerto externo: 443
       Puerto interno: 443
       IP destino:     192.168.0.59
       Protocolo:      TCP

  4. Guardar y reiniciar router si es necesario

  NOTA: Si tu IP publica no es fija, usa un servicio
  de Dynamic DNS (DDNS) como:
    - DuckDNS (duckdns.org) - gratis
    - No-IP (noip.com) - gratis
    - Cloudflare (si ya usas Cloudflare)

  Con DDNS, actualizas automaticamente tu IP publica
  cuando cambie, y tu dominio siempre apunta al lugar
  correcto.


------------------------------------------------------------
  G) SCRIPT DE INSTALACION COMPLETA (PRODUCCION)
------------------------------------------------------------

#!/bin/bash
# install-production.sh
# Ejecutar como root en el servidor

set -e

DOMAIN="project.tuempresa.com"
EMAIL="admin@tuempresa.com"

echo "=== Instalando dependencias ==="
apt update && apt install -y nginx certbot python3-certbot-nginx

echo "=== Configurando Nginx ==="
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

echo "=== Configurando SSL ==="
certbot --nginx -d ${DOMAIN} --non-interactive --agree-tos -m ${EMAIL}

echo "=== Configurando firewall ==="
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow OpenSSH
ufw --force enable

echo "=== Listo ==="
echo "Accede a: https://${DOMAIN}"


------------------------------------------------------------
  H) COMANDOS UTILES PARA PRODUCCION
------------------------------------------------------------

# Ver logs de Nginx
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# Renovar certificado SSL manualmente
sudo certbot renew

# Ver estado del certificado
sudo certbot certificates

# Verificar configuracion de Nginx
sudo nginx -t

# Recargar Nginx sin reiniciar
sudo systemctl reload nginx

# Reiniciar Nginx completo
sudo systemctl restart nginx

# Verificar que OpenProject responde internamente
curl -I http://127.0.0.1:8080

# Verificar que Nginx responde
curl -I http://localhost

# Verificar HTTPS externo
curl -I https://project.tuempresa.com

# Backup de la base de datos (hacer periodicamente)
sudo docker compose exec openproject psql -U openproject -d openproject > backup_$(date +%Y%m%d).sql

# Backup de archivos adjuntos
sudo tar -czf assets_$(date +%Y%m%d).tar.gz /var/lib/openproject/assets


===========================================================
  FIN DE LA GUIA
  Version: OpenProject 17.7 (all-in-one)
  Local: http://192.168.0.59:8080
  Produccion: https://project.tuempresa.com
  Admin: admin / admin (CAMBIAR EN PRIMER INGRESO)
===========================================================
