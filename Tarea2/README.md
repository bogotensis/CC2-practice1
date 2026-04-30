# Tarea 2: Despliegue OwnCloud con Alta Disponibilidad (Escenario 2)

## Nombre del alumno
**Erwin Andrei Hortua Cortes**

---

## Entorno de desarrollo y producción

| Componente | Versión / Detalle |
|---|---|
| Servidor de producción | `docker.ugr.es` |
| Sistema operativo del servidor | Linux (UGR) |
| Gestor de contenedores | Podman |
| Orquestador | podman-compose |
| Puertos asignados | 20160 – 20169 |

---

## Descripción

Despliegue de OwnCloud con alta disponibilidad siguiendo el **Escenario 2** (empresa mediana, hasta 1000 usuarios). Incluye balanceo de carga con HAProxy y replicación del servicio web OwnCloud.

---

## Arquitectura desplegada (Escenario 2)

```
                        ┌─────────────────────────────────────┐
                        │           docker.ugr.es             │
                        │                                     │
  Usuario  ──HTTP──►  :20160                                  │
                        │         ┌──────────────┐            │
                        │         │   HAProxy    │            │
                        │         │  :80→20160   │            │
                        │         └──────┬───────┘            │
                        │         roundrobin                  │
                        │       ┌────────┴────────┐           │
                        │       ▼                 ▼           │
                        │  OwnCloud1          OwnCloud2       │
                        │  :8080              :8080           │
                        │       └────────┬────────┘           │
                        │                │                    │
                        │    ┌───────────┼───────────┐        │
                        │    ▼           ▼           ▼        │
                        │ MariaDB      Redis      OpenLDAP    │
                        │ (interna)  (interna)  :20163/:20164 │
                        │                                     │
                        │  Stats HAProxy: :20165              │
                        └─────────────────────────────────────┘
```

Red interna: `owncloud_net` (bridge).

---

## Servicios desplegados

### 1. HAProxy (`haproxytech/haproxy-alpine:2.4`)

Balanceador de carga que distribuye las peticiones entre las dos réplicas de OwnCloud.

| Parámetro | Valor |
|---|---|
| Puerto externo | `20160` → `80` |
| Puerto stats | `20165` → `8404` |
| Algoritmo | `roundrobin` |
| Backend | `owncloud1:8080`, `owncloud2:8080` |

### 2. OwnCloud réplica 1 y 2 (`owncloud/server:latest`)

Dos instancias del servicio web OwnCloud compartiendo la misma base de datos y volumen de datos.

| Parámetro | Valor |
|---|---|
| Admin usuario | `admin` |
| Admin password | `admin` |
| Dominio | `docker.ugr.es:20160` |
| Persistencia | volumen nombrado `owncloud_data` (compartido) |

### 3. OpenLDAP (`osixia/openldap:1.5.0`)

| Parámetro | Valor |
|---|---|
| Puerto externo LDAP | `20163` → `389` |
| Puerto externo LDAPS | `20164` → `636` |
| Dominio | `dc=example,dc=org` |
| Admin DN | `cn=admin,dc=example,dc=org` |
| Admin password | `admin` |
| Persistencia | volúmenes nombrados `slapd_config`, `slapd_database` |

### 4. MariaDB (`mariadb:latest`)

| Parámetro | Valor |
|---|---|
| Base de datos | `owncloud` |
| Usuario | `owncloud` |
| Contraseña | `owncloud` |
| Root password | `rootpassword` |
| Persistencia | volumen nombrado `mariadb_data` |

### 5. Redis (`redis:latest`)

| Parámetro | Valor |
|---|---|
| Host (interno) | `redis` |
| Puerto (interno) | `6379` |

---

## Instrucciones de despliegue

### 1. Conectarse al servidor y clonar el repositorio

```bash
ssh <usuario>@docker.ugr.es
git clone https://github.com/bogotensis/CC2-practice1.git
cd CC2-practice1/Tarea2
```

### 2. Levantar los servicios

```bash
podman-compose up -d
```

Verificar que los 6 contenedores están corriendo:

```bash
podman ps
```

Se deben ver: `openldap`, `mariadb`, `redis`, `owncloud1`, `owncloud2`, `haproxy`.

### 3. Cargar usuarios LDAP

```bash
podman exec -i openldap ldapadd -x \
  -D "cn=admin,dc=example,dc=org" \
  -w admin < ../Tarea1/users.ldif
```

Establecer contraseñas:

```bash
for uid in ahortua jperez mgarcia; do
  podman exec openldap ldappasswd -s password123 -w admin \
    -D "cn=admin,dc=example,dc=org" -x \
    "uid=$uid,ou=People,dc=example,dc=org"
done
```

### 4. Configurar LDAP en OwnCloud

```bash
podman exec owncloud1 occ app:enable user_ldap
podman exec owncloud1 occ ldap:create-empty-config
podman exec owncloud1 occ ldap:set-config s01 ldapHost openldap
podman exec owncloud1 occ ldap:set-config s01 ldapPort 389
podman exec owncloud1 occ ldap:set-config s01 ldapAgentName "cn=admin,dc=example,dc=org"
podman exec owncloud1 occ ldap:set-config s01 ldapAgentPassword admin
podman exec owncloud1 occ ldap:set-config s01 ldapBase "dc=example,dc=org"
podman exec owncloud1 occ ldap:set-config s01 ldapBaseUsers "ou=People,dc=example,dc=org"
podman exec owncloud1 occ ldap:set-config s01 ldapBaseGroups "ou=Groups,dc=example,dc=org"
podman exec owncloud1 occ ldap:set-config s01 ldapUserFilter "(objectClass=inetOrgPerson)"
podman exec owncloud1 occ ldap:set-config s01 ldapLoginFilter "(&(objectClass=inetOrgPerson)(uid=%uid))"
podman exec owncloud1 occ ldap:set-config s01 ldapUserDisplayName cn
podman exec owncloud1 occ ldap:set-config s01 ldapUserName uid
podman exec owncloud1 occ ldap:set-config s01 ldapConfigurationActive 1
```

Verificar y sincronizar:

```bash
podman exec owncloud1 occ ldap:test-config s01
podman exec owncloud1 occ user:sync "OCA\User_LDAP\User_Proxy" -m disable
podman exec owncloud1 occ user:list
```

### 5. Acceder a OwnCloud y stats de HAProxy

- OwnCloud: `http://docker.ugr.es:20160`
- Stats HAProxy: `http://docker.ugr.es:20165`

### 6. Bajar los servicios

```bash
podman-compose down
```

---

## Verificación del balanceo de carga

Accede a `http://docker.ugr.es:20165` para ver el dashboard de HAProxy y comprobar que las peticiones se reparten entre `owncloud1` y `owncloud2`.

---

## Conclusiones

- HAProxy distribuye las peticiones entre las dos réplicas de OwnCloud con algoritmo roundrobin, proporcionando alta disponibilidad y balanceo de carga.
- Ambas réplicas comparten el mismo volumen de datos y base de datos, garantizando consistencia entre instancias.
- El uso de healthchecks en `docker-compose.yml` evita condiciones de carrera durante la inicialización, asegurando que `owncloud2` solo arranca cuando `owncloud1` ha terminado de inicializar la base de datos.
- Los volúmenes nombrados de Podman resuelven los problemas de permisos propios de entornos rootless.
- La etiqueta `:z` en el volumen de HAProxy es necesaria para que Podman ajuste el contexto SELinux y permita la lectura del fichero de configuración.

---

## Referencias

- HAProxy docs: http://docs.haproxy.org/2.6/configuration.html
- HAProxy load balancing: https://www.haproxy.com/blog/haproxy-configuration-basics-load-balance-your-servers/
- OwnCloud occ CLI: https://doc.owncloud.com/server/next/admin_manual/configuration/server/occ_command.html
- Podman rootless volumes: https://docs.podman.io/en/latest/markdown/podman-volume.1.html
