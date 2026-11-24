# nas-apps

Stacks que corren en el TrueNAS, en su propio swarm (el NAS es el único manager, label `ubicacion=nas`),
manejados por un swarmdeck que corre en el mismo NAS.

## swarmdeck: el swarm propio del NAS

### 1. Si el NAS es worker de otro swarm: salir

1. En el swarmdeck de ese swarm, guardar lo elegido para los stacks del NAS (variables, "Seguir imagen",
   usuario y token del repo): no viaja solo. Después sacarlos de su `stacks.yml`, reiniciarlo y, en ese
   manager, `docker stack rm holyarr`.
2. En el NAS, como root: `docker swarm leave`. No toca las Apps de TrueNAS (Tailscale incluida), SSH,
   usuarios, datasets, carpetas ni volúmenes.
3. En el manager viejo, cuando figure `Down`: `docker node rm NOMBRE_DEL_NAS`. No usar la baja de nodo de
   swarmdeck: también lo saca de la VPN. Si el NAS es destino de backups (`node:...`), tiene
   que seguir en el inventario (`nodes.yml`) de ese swarmdeck: los backups van por SSH, no por el swarm.

### 2. Crear el swarm

```bash
docker swarm init --advertise-addr IP_LAN_DEL_NAS
docker node update --label-add ubicacion=nas "$(docker info -f '{{.Swarm.NodeID}}')"
```

La IP de la red local (fija o reservada en el router), no la de Tailscale: si Tailscale corre como App,
arranca después que Docker, y al prender el NAS esa IP todavía no existe.

### 3. Instalar swarmdeck

Copiar `swarmdeck-X.Y.Z-linux.tar.gz` y `swarmdeck/docker-stack.yml` a `/mnt/HomePool/swarmdeck-instalar`
y, como root:

```bash
cd /mnt/HomePool/swarmdeck-instalar
ss -ltn | grep ':8000 '                      # no tiene que mostrar nada (si no, cambiar published)
tar xzf swarmdeck-0.37.0-linux.tar.gz
docker build -t swarmdeck:0.37.0 swarmdeck-0.37.0
mkdir -p /mnt/HomePool/swarmdeck && chmod 700 /mnt/HomePool/swarmdeck
( umask 077; printf '%s' "$(openssl rand -hex 32)" > token )
docker secret create swarmdeck_token token
docker stack deploy --resolve-image never -c docker-stack.yml swarmdeck
cat token; echo                              # para entrar; guardarlo y después: rm token
```

Entrar en `http://IP_LAN_DEL_NAS:8000` (o la de Tailscale). Nunca abrir el 8000 a internet.

Para actualizar: construir la imagen nueva, cambiar `image:` y repetir el `docker stack deploy`.

## holyarr: HolyArrs + SABnzbd (Usenet) + qBittorrent (torrents) + Jellyfin

`IP_DEL_NAS`: en casa, la del NAS en la red local; desde afuera, la de Tailscale.

### 1. Lo que hace falta afuera

- **Un proveedor de Usenet** (pago, por mes o por bloque de GB): te da servidor, puerto SSL (563),
  usuario y contraseña. Conviene uno con buena retención y conexiones SSL.
- **Uno o más indexadores** (sitios que dicen qué hay en Usenet): te dan una URL y una API key para
  cargar en HolyArrs como Sources.

### 2. En el TrueNAS (una vez)

```bash
id apps        # uid y gid: si no son 568, cambialos en docker-stack.yml (PUID/PGID)
sudo mkdir -p /mnt/HomePool/holyarr/config/holyarrs /mnt/HomePool/holyarr/config/sabnzbd \
  /mnt/HomePool/holyarr/config/qbittorrent /mnt/HomePool/holyarr/config/jellyfin \
  /mnt/HomePool/holyarr/data/downloads/incomplete /mnt/HomePool/holyarr/data/downloads/complete \
  /mnt/HomePool/holyarr/data/downloads/torrents \
  /mnt/HomePool/holyarr/data/movies /mnt/HomePool/holyarr/data/tv /mnt/HomePool/holyarr/data/music
sudo chown -R apps:apps /mnt/HomePool/holyarr
```

Swarm no crea las carpetas de los volúmenes: si falta una, el servicio no arranca ("bind source path does
not exist").

Si el dataset tiene permisos tipo ACL, dale el dueño `apps` desde Datasets → holyarr → Permissions.

### 3. En swarmdeck

1. **Cluster → el NAS**: tiene que estar disponible y con la label `ubicacion=nas`.
2. **Instalación → Stacks**: agregar este repo como `holyarr`, con `holyarr/docker-stack.yml` y la casilla
   de confianza (monta carpetas del NAS). Después, reiniciar swarmdeck desde ahí.
3. **Sincronizar** `holyarr`.

### 4. SABnzbd (http://IP_DEL_NAS:8080)

1. El asistente pide el servidor de Usenet: el del proveedor, SSL, puerto 563.
2. **Config → Folders**: temporales `/data/downloads/incomplete`, terminados `/data/downloads/complete`.
   HolyArrs ya sabe que lo terminado queda ahí (`HOLYARRS_SAVE_PATH` en el stack): si cambiás esta
   carpeta, cambiá también esa variable.
3. **Config → Special → host_whitelist**: agregar `sabnzbd` (así HolyArrs puede llamarlo por nombre).
4. **Config → General**: copiar la **API Key**.

### 5. qBittorrent (http://IP_DEL_NAS:8081)

1. La primera vez genera una contraseña temporal para `admin`: está en los logs del servicio
   (swarmdeck → Stacks → holyarr → Logs de `qbittorrent`, buscar "temporary password").
   Entrá y cambiala en **Tools → Options → Web UI**.
2. **Options → Downloads**: guardar en `/data/downloads/torrents`.
3. **Options → Connection**: puerto `6881`. Para que bajen bien, abrí el 6881 TCP y UDP en el router de
   casa hacia el NAS (es el único puerto que va a internet; la interfaz web nunca).
4. **Options → BitTorrent**: conviene limitar el ratio o el tiempo de seed.

Ojo: por torrent, los demás ven la IP pública de tu casa. Si eso te importa, se puede pasar qBittorrent
por una VPN comercial (con gluetun); preguntá antes de hacerlo.

### 6. HolyArrs (http://IP_DEL_NAS:6262)

1. **Entrar enseguida**: el primero que entra crea la cuenta de dueño.
2. Bibliotecas: `/data/movies`, `/data/tv`, `/data/music`.
3. Descargadores: SABnzbd en `http://sabnzbd:8080` con la API key, y qBittorrent en
   `http://qbittorrent:8081` con su usuario y contraseña.
4. Sources: los indexadores, con su API key.

### 7. Jellyfin (http://IP_DEL_NAS:8096)

1. El asistente pide crear el usuario administrador.
2. Bibliotecas: Películas en `/data/movies`, Series en `/data/tv`, Música en `/data/music` (las ve solo
   lectura: Jellyfin no mueve ni borra archivos).
3. En los televisores y teléfonos, la app de Jellyfin lo encuentra sola en casa (puerto 7359/udp); si no,
   `http://IP_DEL_NAS:8096`.
4. En HolyArrs, si ofrece avisar a Jellyfin al terminar una descarga: `http://jellyfin:8096` con una API key
   de Jellyfin (Panel → Claves API).

Sin transcodificación por hardware: Swarm no pasa la GPU del NAS a los servicios. Con apps que reproducen el
archivo tal cual (lo normal en casa) no hace falta; si se necesita, Jellyfin como App de TrueNAS.

Nunca abrir 6262, 8080, 8081 ni 8096 a internet: en casa por la red local, desde afuera por Tailscale. Solo
el 6881 (torrents) va al router.
