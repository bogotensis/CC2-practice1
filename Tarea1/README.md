# Práctica 1: Despliegue de servicio OwnCloud con contenedores

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

## Descripción de la práctica y problema a resolver

El objetivo de esta práctica es desplegar un servicio auto-alojado de **OwnCloud** (alternativa open-source a Google Drive o Dropbox) usando tecnologías de contenedores, siguiendo la arquitectura del **Escenario 1** (pequeña empresa, hasta 150 usuarios).

El problema central es poner en marcha un conjunto de microservicios interconectados que, de forma conjunta, ofrezcan:
- Almacenamiento y gestión de ficheros en la nube (OwnCloud).
- Persistencia de datos mediante una base de datos relacional (MariaDB).
- Caché y bloqueo de ficheros para mejorar el rendimiento (Redis).
- Autenticación centralizada de usuarios mediante directorio (OpenLDAP).

Todo el despliegue se realiza con `podman-compose`, garantizando que los datos persisten más allá del ciclo de vida de los contenedores mediante volúmenes nombrados de Podman (en lugar de bind mounts, para compatibilidad con entornos rootless).

---

## Arquitectura desplegada (Escenario 1)

```
                        ┌─────────────────────────────────────┐
                        │           docker.ugr.es             │
                        │                                     │
  Usuario  ──HTTP──►  :20160                                  │
                        │         ┌──────────────┐            │
                        │         │   OwnCloud   │            │
                        │         │  :8080→20160 │            │
                        │         └──────┬───────┘            │
                        │                │                    │
                        │    ┌───────────┼───────────┐        │
                        │    ▼           ▼           ▼        │
                        │ MariaDB      Redis      OpenLDAP    │
                        │ (interna)  (interna)  :20163/:20164 │
                        └─────────────────────────────────────┘
```

Red interna: `owncloud_net` (bridge). Solo OwnCloud y OpenLDAP exponen puertos al exterior.

---

## Servicios desplegados y configuración

### 1. OpenLDAP (`osixia/openldap:1.5.0`)

Servicio de autenticación de usuarios mediante el protocolo LDAP.

| Parámetro | Valor |
|---|---|
| Puerto externo LDAP | `20163` → `389` |
| Puerto externo LDAPS | `20164` → `636` |
| Dominio | `dc=example,dc=org` |
| Admin DN | `cn=admin,dc=example,dc=org` |
| Admin password | `admin` |
| Persistencia config | volumen nombrado `slapd_config` |
| Persistencia datos | volumen nombrado `slapd_database` |

**Estructura del directorio LDAP:**
```
dc=example,dc=org
├── ou=People          ← usuarios de la organización
│   ├── uid=ahortua
│   ├── uid=jperez
│   └── uid=mgarcia
└── ou=Groups          ← grupos
    └── cn=employees
```

### 2. MariaDB (`mariadb:latest`)

Base de datos relacional que almacena todos los metadatos de OwnCloud.

| Parámetro | Valor |
|---|---|
| Base de datos | `owncloud` |
| Usuario | `owncloud` |
| Contraseña | `owncloud` |
| Root password | `rootpassword` |
| Persistencia | volumen nombrado `mariadb_data` |
| Red | interna (`owncloud_net`) |

### 3. Redis (`redis:latest`)

Caché en memoria para bloqueo de ficheros y mejora de rendimiento de OwnCloud.

| Parámetro | Valor |
|---|---|
| Host (interno) | `redis` |
| Puerto (interno) | `6379` |
| Red | interna (`owncloud_net`) |

### 4. OwnCloud (`owncloud/server:latest`)

Aplicación principal de almacenamiento y gestión de ficheros.

| Parámetro | Valor |
|---|---|
| Puerto externo | `20160` → `8080` |
| Admin usuario | `admin` |
| Admin password | `admin` |
| Dominio | `docker.ugr.es:20160` |
| Persistencia | volumen nombrado `owncloud_data` |

---

## Instrucciones de despliegue

### Requisitos previos

Conectarse al servidor:
```bash
ssh <usuario>@docker.ugr.es
```

Clonar el repositorio:
```bash
git clone https://github.com/bogotensis/CC2.git
cd CC2/cc2526/practice1
```

### 1. Levantar todos los servicios

```bash
podman-compose up -d
```

Verificar que todos los contenedores están en ejecución:
```bash
podman ps
```

Se deben ver 4 contenedores: `openldap`, `mariadb`, `redis`, `owncloud`.

### 2. Cargar usuarios en LDAP

Esperar unos segundos a que OpenLDAP arranque completamente, luego:

```bash
podman exec -i openldap ldapadd -x \
  -D "cn=admin,dc=example,dc=org" \
  -w admin < users.ldif
```

Verificar que los usuarios se han creado:
```bash
podman exec openldap ldapsearch -x \
  -H ldap://localhost:389 \
  -b dc=example,dc=org \
  -D "cn=admin,dc=example,dc=org" \
  -w admin
```

Establecer contraseñas para los usuarios:
```bash
```bash
for uid in ahortua jperez mgarcia; do
  podman exec openldap ldappasswd -s password123 -w admin \
    -D "cn=admin,dc=example,dc=org" -x \
    "uid=$uid,ou=People,dc=example,dc=org"
done
```

### 3. Acceder a OwnCloud

Abrir en el navegador:
```
http://docker.ugr.es:20160
```

Credenciales de administrador: `admin` / `admin`

### 4. Integrar LDAP en OwnCloud

1. Ir a **Apps → Herramientas** y activar **LDAP Integration**
2. Configurar via consola:

```bash
podman exec owncloud occ ldap:create-empty-config
podman exec owncloud occ ldap:set-config s01 ldapHost openldap
podman exec owncloud occ ldap:set-config s01 ldapPort 389
podman exec owncloud occ ldap:set-config s01 ldapAgentName "cn=admin,dc=example,dc=org"
podman exec owncloud occ ldap:set-config s01 ldapAgentPassword admin
podman exec owncloud occ ldap:set-config s01 ldapBase "dc=example,dc=org"
podman exec owncloud occ ldap:set-config s01 ldapBaseUsers "ou=People,dc=example,dc=org"
podman exec owncloud occ ldap:set-config s01 ldapBaseGroups "ou=Groups,dc=example,dc=org"
podman exec owncloud occ ldap:set-config s01 ldapUserFilter "(objectClass=inetOrgPerson)"
podman exec owncloud occ ldap:set-config s01 ldapLoginFilter "(&(objectClass=inetOrgPerson)(uid=%uid))"
podman exec owncloud occ ldap:set-config s01 ldapUserDisplayName cn
podman exec owncloud occ ldap:set-config s01 ldapUserName uid
podman exec owncloud occ ldap:set-config s01 ldapConfigurationActive 1
```

Verificar que la configuración es válida:
```bash
podman exec owncloud occ ldap:test-config s01
```

Sincronizar usuarios:
```bash
podman exec owncloud occ user:sync "OCA\User_LDAP\User_Proxy" -m disable
```

### 5. Bajar los servicios

```bash
podman-compose down
```

Los datos persisten en los volúmenes nombrados de Podman y estarán disponibles al volver a levantar los servicios.

---

## Verificación de persistencia

Para comprobar que los datos persisten tras reiniciar los contenedores:

```bash
podman-compose down
podman-compose up -d
podman exec owncloud occ user:list
```

---

## Conclusiones

- El uso de `podman-compose` simplifica enormemente el despliegue de arquitecturas multi-contenedor, permitiendo definir todos los servicios, sus dependencias, variables de entorno y volúmenes en un único fichero declarativo.
- La persistencia de datos es un aspecto crítico. En entornos Podman rootless, los bind mounts pueden causar problemas de permisos, por lo que se recomienda usar volúmenes nombrados que Podman gestiona internamente.
- La integración de LDAP con OwnCloud permite centralizar la gestión de usuarios, lo que es especialmente útil en entornos organizacionales donde los usuarios ya están gestionados en un directorio corporativo.
- Redis mejora significativamente el rendimiento de OwnCloud al gestionar el bloqueo de ficheros en memoria en lugar de en base de datos.
- La red bridge interna (`owncloud_net`) aísla los servicios del exterior, exponiendo únicamente los puertos estrictamente necesarios.

---

## Referencias bibliográficas y recursos utilizados

- Enunciado de la práctica: https://github.com/j-m-benitez/cc2526/tree/main/practice1
- Documentación oficial OwnCloud: https://doc.owncloud.com/server/next/admin_manual/
- Recomendaciones de despliegue OwnCloud: https://doc.owncloud.com/server/next/admin_manual/installation/deployment_recommendations.html
- Imagen Docker OpenLDAP: https://github.com/osixia/docker-openldap
- Documentación OpenLDAP: https://www.openldap.org/doc/admin26/
- Integración LDAP con OwnCloud: https://doc.owncloud.com/server/next/admin_manual/configuration/user/user_auth_ldap.html
- MariaDB Docker Hub: https://hub.docker.com/_/mariadb
- Redis Docker Hub: https://hub.docker.com/_/redis
- Introducción a HAProxy: http://docs.haproxy.org/2.6/intro.html
- Podman rootless volumes: https://docs.podman.io/en/latest/markdown/podman-volume.1.html
- OwnCloud occ CLI: https://doc.owncloud.com/server/next/admin_manual/configuration/server/occ_command.html
