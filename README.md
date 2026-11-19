# nas-apps

Stacks que corren en el TrueNAS (worker del swarm, label `ubicacion=nas`), manejados por swarmdeck.

## holyarr: HolyArrs + SABnzbd (Usenet)

### 1. Lo que hace falta afuera

- **Un proveedor de Usenet** (pago, por mes o por bloque de GB): te da servidor, puerto SSL (563),
  usuario y contraseña. Conviene uno con buena retención y conexiones SSL.
- **Uno o más indexadores** (sitios que dicen qué hay en Usenet): te dan una URL y una API key para
  cargar en HolyArrs como Sources.

### 2. En el TrueNAS (una vez)

```bash
id apps        # uid y gid: si no son 568, cambialos en docker-stack.yml (PUID/PGID)
sudo mkdir -p /mnt/HomePool/holyarr/config/{holyarrs,sabnzbd} \
              /mnt/HomePool/holyarr/data/{downloads/incomplete,downloads/complete,movies,tv,music}
sudo chown -R apps:apps /mnt/HomePool/holyarr
```

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

### 5. HolyArrs (http://IP_TAILSCALE_DEL_NAS:6262)

1. **Entrar enseguida**: el primero que entra crea la cuenta de dueño.
2. Bibliotecas: `/data/movies`, `/data/tv`, `/data/music`.
3. Descargador: SABnzbd en `http://sabnzbd:8080` con la API key.
4. Sources: los indexadores, con su API key.

Nunca abrir 6262 ni 8080 a internet: se entra por Tailscale.
