# nas-apps

Stacks que corren en el TrueNAS (worker del swarm, label `ubicacion=nas`), manejados por swarmdeck.

## holyarr: HolyArrs + SABnzbd (Usenet) + qBittorrent (torrents)

### 1. Lo que hace falta afuera

- **Un proveedor de Usenet** (pago, por mes o por bloque de GB): te da servidor, puerto SSL (563),
  usuario y contraseña. Conviene uno con buena retención y conexiones SSL.
- **Uno o más indexadores** (sitios que dicen qué hay en Usenet): te dan una URL y una API key para
  cargar en HolyArrs como Sources.

### 2. En el TrueNAS (una vez)

```bash
id apps        # uid y gid: si no son 568, cambialos en docker-stack.yml (PUID/PGID)
sudo mkdir -p /mnt/HomePool/holyarr/config/holyarrs /mnt/HomePool/holyarr/config/sabnzbd \
  /mnt/HomePool/holyarr/config/qbittorrent \
  /mnt/HomePool/holyarr/data/downloads/incomplete /mnt/HomePool/holyarr/data/downloads/complete \
  /mnt/HomePool/holyarr/data/downloads/torrents \
  /mnt/HomePool/holyarr/data/movies /mnt/HomePool/holyarr/data/tv /mnt/HomePool/holyarr/data/music
sudo chown -R apps:apps /mnt/HomePool/holyarr
```

Swarm no crea las carpetas de los volúmenes: si falta una, el servicio no arranca ("bind source path does
not exist").

Si el dataset tiene permisos tipo ACL, dale el dueño `apps` desde Datasets → holyarr → Permissions.

### 3. En swarmdeck

1. **Red → Probar red overlay**: el NAS tiene que dar "anda".
2. **Cluster → el NAS**: Terminar mantenimiento y **Labels… → `ubicacion=nas`**.
3. Este repo en GitHub (privado) → **Instalación → Stacks**: agregar `holyarr` con
   `holyarr/docker-stack.yml`.
4. Como monta carpetas del NAS, el stack tiene que ser de confianza: en el VPS, en
   `/var/lib/swarmdeck/stacks.yml`, agregar `trusted: true` a su entrada y
   `sudo systemctl restart swarmdeck`.
5. **Sincronizar** `holyarr`.

### 4. SABnzbd (http://IP_TAILSCALE_DEL_NAS:8080)

1. El asistente pide el servidor de Usenet: el del proveedor, SSL, puerto 563.
2. **Config → Folders**: temporales `/data/downloads/incomplete`, terminados `/data/downloads/complete`.
3. **Config → Special → host_whitelist**: agregar `sabnzbd` (así HolyArrs puede llamarlo por nombre).
4. **Config → General**: copiar la **API Key**.

### 5. qBittorrent (http://IP_TAILSCALE_DEL_NAS:8081)

1. La primera vez genera una contraseña temporal para `admin`: está en los logs del servicio
   (swarmdeck → Stacks → holyarr → Logs de `qbittorrent`, buscar "temporary password").
   Entrá y cambiala en **Tools → Options → Web UI**.
2. **Options → Downloads**: guardar en `/data/downloads/torrents`.
3. **Options → Connection**: puerto `6881`. Para que bajen bien, abrí el 6881 TCP y UDP en el router de
   casa hacia el NAS (es el único puerto que va a internet; la interfaz web nunca).
4. **Options → BitTorrent**: conviene limitar el ratio o el tiempo de seed.

Ojo: por torrent, los demás ven la IP pública de tu casa. Si eso te importa, se puede pasar qBittorrent
por una VPN comercial (con gluetun); preguntá antes de hacerlo.

### 6. HolyArrs (http://IP_TAILSCALE_DEL_NAS:6262)

1. **Entrar enseguida**: el primero que entra crea la cuenta de dueño.
2. Bibliotecas: `/data/movies`, `/data/tv`, `/data/music`.
3. Descargadores: SABnzbd en `http://sabnzbd:8080` con la API key, y qBittorrent en
   `http://qbittorrent:8081` con su usuario y contraseña.
4. Sources: los indexadores, con su API key.

Nunca abrir 6262, 8080 ni 8081 a internet: se entra por Tailscale. Solo el 6881 (torrents) va al router.
