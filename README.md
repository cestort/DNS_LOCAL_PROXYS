# Docker DNS + Reverse Proxy para la LAN

Infraestructura local con **Pi-hole** (DNS con bloqueo de anuncios) y **Nginx Proxy Manager** (reverse proxy con panel web) para acceder a los servicios desplegados mediante nombres `.local` en lugar de IP:puerto.

## Servicios del stack

| Servicio | Contenedor | Imagen | Puerto host | Función |
|----------|------------|--------|-------------|---------|
| Pi-hole | `pihole` | `pihole/pihole:latest` | 53 (TCP/UDP), 8080 | DNS de la LAN + bloqueo de anuncios |
| Nginx Proxy Manager | `nginx-proxy` | `jc21/nginx-proxy-manager:latest` | 80, 81 | Reverse proxy + panel web |

### Pi-hole (DNS) — `http://192.168.5.112:8080/admin`

- DNS local para toda la LAN en el puerto 53.
- Reenvía consultas desconocidas a servidores externos (`8.8.8.8`), no bloquea el acceso a internet.
- El wildcard `*.local` (archivo `etc-dnsmasq.d/local-wildcard.conf`) hace que **cualquier** `nombre.local` resuelva al IP del reverse proxy.
- Cambios de DNS locales se hacen en el panel: **Local DNS → DNS Records**.

> Importante: si recreas el contenedor, la contraseña de Pi-hole v6 se maneja dentro del contenedor (`pihole setpassword nueva-clave`). El `WEBPASSWORD` del compose no se aplica en v6.

### Nginx Proxy Manager (reverse proxy) — `http://192.168.5.112:81`

- Cede el puerto 80 (HTTP) para enrutar los dominios `.local`.
- Panel de administración en el puerto 81 (setup inicial crea el usuario admin).
- Cada servicio se registra en **Hosts → Proxy Hosts** apuntando a su IP:puerto interno.

## Red y DNS

- IP del host: `192.168.5.112`.
- Los dispositivos deben usar Pi-hole como DNS para resolver los dominios `.local`. En Windows, preferido `192.168.5.112` y alternativo `8.8.8.8`.
- Servidor final accesible en `django.local`, `traductor.local`, `whisper.local`, etc., sin puerto.

## Ejemplo: exponer la aplicación Django "argos" como `django.local`

App corriendo en el host en el puerto `8009`. Pasos completos:

1. **Verificar que la app responde y escucha en todas las interfaces:**

   ```powershell
   Invoke-WebRequest http://192.168.5.112:8009   # HTTP 200
   Get-NetTCPConnection -LocalPort 8009 -State Listen  # LocalAddress 0.0.0.0
   ```

2. **Comprobar resolución DNS del nombre** (lo aporta el wildcard de Pi-hole):

   ```powershell
   nslookup django.local 192.168.5.112
   # Nombre: django.local -> Address: 192.168.5.112
   ```

3. **Configurar el DNS del equipo** para que resuelva `.local` (PowerShell como administrador):

   ```powershell
   Set-DnsClientServerAddress -InterfaceIndex 6 -ServerAddresses ("192.168.5.112","8.8.8.8")
   ```

4. **Crear el Proxy Host en NPM** (`http://192.168.5.112:81`):
   - **Hosts → Proxy Hosts → Add Proxy Host**
   - Domain Names: `django.local`
   - Scheme: `http` · Forward Hostname: `192.168.5.112` · Forward Port: `8009`
   - Save.

5. **Acceder por nombre:**

   `http://django.local` → HTTP 200.

## Comandos útiles

```bash
docker compose up -d          # levantar el stack
docker compose down           # detener (conserva los datos en las carpetas)
docker compose ps             # estado
docker logs pihole -f         # logs de DNS
docker logs nginx-proxy -f    # logs del proxy
```

## Datos persistidos

| Carpeta | Contenido |
|---------|-----------|
| `etc-pihole/` | Configuración y base de datos de Pi-hole |
| `etc-dnsmasq.d/` | Registros DNS locales (`*.local`) |
| `npm-data/` | Proxy hosts y configuración de NPM |
| `npm-letsencrypt/` | Certificados de NPM |