# Desplegar Postiz en Coolify

Usa `docker-compose.coolify.yaml` (no el `docker-compose.yaml` original, que se deja intacto para poder sincronizar con upstream).

## Requisitos
- VPS con Coolify v4, mínimo 2 vCPU / 2 GB RAM libres (recomendado 4 GB).
- El subdominio `mk.creapps.co` con registro A apuntando a la IP del VPS.

## 1. Crear el recurso
1. Coolify → Project → **+ New Resource** → repositorio de GitHub (GitHub App o repo público) → `creappsco/postiz-docker-compose`, rama `main`.
2. **Build Pack:** `Docker Compose`.
3. **Docker Compose Location:** `/docker-compose.coolify.yaml`.
4. Pulsa **Load Compose File**.

## 2. Dominio
En el servicio **postiz**, campo *Domains*:

```
https://mk.creapps.co:5000
```

**El `:5000` no aparece en la URL pública.** Es solo la forma en que Coolify sabe a qué puerto *dentro del contenedor* mandar el tráfico. Los usuarios entran por `https://mk.creapps.co` (puerto 443, SSL automático) y Traefik lo reenvía internamente al 5000. El puerto 5000 no se abre en el VPS.

Si escribes el dominio sin `:5000`, Coolify asume el puerto 80 dentro del contenedor y Postiz da *Bad Gateway*. Los demás servicios van sin dominio.

## 3. Variables de entorno (pestaña *Environment Variables*)

| Variable | Obligatoria | Valor |
|---|---|---|
| `POSTIZ_URL` | sí | `https://mk.creapps.co` (sin `/` final, sin puerto) |
| `JWT_SECRET` | sí | `openssl rand -hex 32` |
| `POSTIZ_DB_PASSWORD` | sí | `openssl rand -hex 24` |
| `TEMPORAL_DB_PASSWORD` | sí | `openssl rand -hex 24` |
| `DISABLE_REGISTRATION` | no | `false` hasta crear tu cuenta, luego `true` |
| `POSTIZ_VERSION` | no | `latest` por defecto; fija un tag de [ghcr.io/gitroomhq/postiz-app](https://github.com/gitroomhq/postiz-app/pkgs/container/postiz-app) para evitar sorpresas |
| `ES_JAVA_OPTS` | no | `-Xms256m -Xmx256m` por defecto |
| `X_API_KEY`, `LINKEDIN_CLIENT_ID`, … | no | Credenciales de cada red social |

Usa contraseñas en hexadecimal: `POSTIZ_DB_PASSWORD` va dentro de una URL de conexión y caracteres como `@`, `/` o `#` la rompen.

> Las contraseñas de las bases de datos se fijan en el **primer** arranque. Si las cambias después, Postgres seguirá usando la antigua (está guardada en el volumen).

## 4. Desplegar
Pulsa **Deploy**. El primer arranque tarda 3–5 min (Elasticsearch y Temporal se inicializan; `postiz` espera a que Temporal esté sano). Luego abre `https://mk.creapps.co` y crea tu cuenta.

Después pon `DISABLE_REGISTRATION=true` y **Redeploy** (un simple restart no aplica variables nuevas).

## 5. Redes sociales
Cada red necesita su app de desarrollador. URL de callback:

```
https://mk.creapps.co/integrations/social/<red>
```

(ej. `.../integrations/social/linkedin`). Tras añadir las claves, Redeploy.

## 6. Backups
Configura backups programados en Coolify o en el VPS para los volúmenes `postgres-volume` (datos de Postiz) y `postiz-uploads` (archivos subidos). Temporal se puede regenerar.

## Problemas frecuentes
- **Login falla / pantalla en blanco:** revisa que `POSTIZ_URL` use `https` y no tenga `/` final.
- **Los posts no se publican:** logs del servicio `temporal`.
- **Elasticsearch se reinicia (exit 137):** falta RAM en el VPS.
- **Temporal no arranca por `dynamicconfig`:** el recurso debe crearse desde Git (paso 1), no pegando el YAML a mano.

## Diferencias con el compose original
- Sin `ports`: el tráfico entra por el proxy de Coolify.
- Sin `container_name` ni redes con nombre fijo (evita choques con otros proyectos).
- Secretos y URL desde variables de entorno, no escritos en el repo.
- Eliminados `spotlight`, `temporal-admin-tools` y `temporal-ui` (no necesarios en producción, ahorran RAM). Si quieres la UI de Temporal, añádela con dominio propio y protección por contraseña.
