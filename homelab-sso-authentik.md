# 🔐 Guía de integración SSO con Authentik — Homelab

**Fecha:** Abril 2026
**Estado:** Documento de referencia para la configuración del SSO en el homelab
**Ya configurados:** Homarr, Open WebUI

---

## 📋 Índice de servicios

| Servicio | Método | Estado |
|---|---|---|
| [Proxmox VE](#1-proxmox-ve) | OIDC nativo | 🔲 Pendiente |
| [Proxmox Backup Server](#2-proxmox-backup-server) | OIDC nativo | 🔲 Pendiente |
| [Vaultwarden](#3-vaultwarden) | OIDC nativo | 🔲 Pendiente |
| [Kavita](#4-kavita) | OIDC nativo | 🔲 Pendiente |
| [Homebox](#5-homebox) | OIDC nativo | 🔲 Pendiente |
| [Home Assistant](#6-home-assistant) | OIDC vía HACS (custom integration) | 🔲 Pendiente |
| [Navidrome](#7-navidrome) | Proxy Forward Auth + cabecera HTTP | 🔲 Pendiente |
| [AdGuard Home](#8-adguard-home) | Proxy Forward Auth | 🔲 Pendiente |
| [metube](#9-metube) | Proxy Forward Auth | 🔲 Pendiente |
| [TriliumNext](#10-triliumnext) | Proxy Forward Auth (OIDC nativo con bugs) | ⚠️ Limitaciones |
| [MyTabs](#11-mytabs--arcane) | Proxy Forward Auth | 🔲 Pendiente |
| [Arcane](#11-mytabs--arcane) | Proxy Forward Auth | 🔲 Pendiente |

---

## Pasos previos comunes

Antes de configurar cualquier servicio, necesitarás crear en Authentik una **Application** y su **Provider** correspondiente. El flujo general es:

1. En Authentik → **Applications → Providers → Create → OAuth2/OpenID Provider**
2. Rellenar los datos del proveedor (redirect URI, scopes, etc.)
3. En **Applications → Create** → asociar al provider creado
4. Anotar el **Client ID**, **Client Secret** e **Issuer URL** del provider

El **Issuer URL** de Authentik para un proveedor con slug `mi-app` siempre tiene el formato:

```
https://TU_AUTHENTIK/application/o/mi-app/
```

---

## 1. Proxmox VE

**Método:** OIDC nativo — Realm OpenID Connect
**Documentación oficial:** [pve.proxmox.com/wiki/OpenID_Connect_SSO](https://pve.proxmox.com/wiki/OpenID_Connect_SSO) | [integrations.goauthentik.io/infrastructure/proxmox/](https://integrations.goauthentik.io/infrastructure/proxmox/)

> ⚠️ **Importante:** Proxmox OIDC no crea usuarios automáticamente. Debes crear el usuario manualmente en Proxmox (`usuario@nombre-realm`) y asignarle permisos antes o después del primer login.

### Paso 1 — Crear Application en Authentik

- **Provider type:** OAuth2/OpenID Connect
- **Client type:** Confidential
- **Redirect URI:**
```
https://TU_PROXMOX:8006/api2/oauth2/callback
```
- **Scopes:** `openid profile email`

### Paso 2 — Configurar el Realm en Proxmox VE

Ir a **Datacenter → Permissions → Realms → Add → OpenID Connect**

| Campo | Valor |
|---|---|
| Realm | `authentik` (o el nombre que prefieras) |
| Issuer URL | `https://TU_AUTHENTIK/application/o/proxmox-ve/` |
| Client ID | El Client ID de Authentik |
| Client Secret | El Client Secret de Authentik |
| Username Claim | `preferred_username` |
| Scopes | `openid profile email` |
| Auto create users | Desactivado (recomendado) |

### Paso 3 — Crear el usuario en Proxmox

En **Datacenter → Permissions → Users → Add**:
- Username: `tu_usuario@authentik`
- Realm: el que acabas de crear

Luego asignar permisos en **Datacenter → Permissions → Add → User Permission**.

### Paso 4 — Verificar

En el login de Proxmox, seleccionar el realm `authentik` en el desplegable y hacer clic en **Login**.

---

## 2. Proxmox Backup Server

**Método:** OIDC nativo — idéntico a Proxmox VE
**Documentación oficial:** [pbs.proxmox.com/docs/user-management.html](https://pbs.proxmox.com/docs/user-management.html#realms)

> El proceso es prácticamente igual que para Proxmox VE. Las diferencias son el puerto (8007 en lugar de 8006) y la URL de callback.

### Paso 1 — Crear Application en Authentik

- **Redirect URI:**
```
https://TU_PBS:8007/api2/oauth2/callback
```
- Resto igual que en Proxmox VE.

### Paso 2 — Configurar el Realm en PBS

Ir a **Datacenter → Permissions → Realms → Add → OpenID Connect**

| Campo | Valor |
|---|---|
| Realm | `authentik` |
| Issuer URL | `https://TU_AUTHENTIK/application/o/proxmox-bs/` |
| Client ID | Client ID de Authentik |
| Client Secret | Client Secret de Authentik |
| Username Claim | `preferred_username` |
| Scopes | `openid profile email` |
|
### Paso 3 — Crear usuario y asignar permisos

Igual que en Proxmox VE: crear el usuario `tu_usuario@authentik` y asignar permisos en **Access Control**.

---

## 3. Vaultwarden

**Método:** OIDC nativo mediante variables de entorno
**Documentación oficial:** [github.com/dani-garcia/vaultwarden/wiki/Enabling-SSO-support-using-OpenId-Connect](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-SSO-support-using-OpenId-Connect)
**Integración Authentik oficial:** [integrations.goauthentik.io/security/vaultwarden/](https://integrations.goauthentik.io/security/vaultwarden/)

> ⚠️ **Requisito:** Vaultwarden debe estar en una versión reciente que incluya SSO. Verifica en la wiki que tu versión lo soporta. La URL de callback es fija y se genera a partir de la variable `DOMAIN`.

### Paso 1 — Crear Application en Authentik

- **Client type:** Confidential
- **Redirect URI:**
```
https://TU_VAULTWARDEN/identity/connect/oidc-signin
```
- **Scopes:** `openid profile email`
- En el provider, asegúrate de que el claim `email_verified` esté incluido en el ID token.

### Paso 2 — Configurar variables de entorno en Docker Compose

```yaml
environment:
  DOMAIN: "https://TU_VAULTWARDEN"
  SSO_ENABLED: "true"
  SSO_ONLY: "false"                          # true = deshabilita login local
  SSO_AUTHORITY: "https://TU_AUTHENTIK/application/o/vaultwarden/"
  SSO_CLIENT_ID: "TU_CLIENT_ID"
  SSO_CLIENT_SECRET: "TU_CLIENT_SECRET"
  SSO_SCOPES: "openid profile email"
  SSO_SIGNUPS_MATCH_EMAIL: "true"
  SSO_ALLOW_UNKNOWN_EMAIL_VERIFICATION: "false"
```

> 💡 Con `SSO_ONLY=false` los usuarios pueden seguir usando su master password. Activa `SSO_ONLY=true` solo si quieres forzar el SSO para todos.

### Paso 3 — Reiniciar y verificar

Tras reiniciar el contenedor, en la pantalla de login aparecerá el botón para acceder vía SSO.

---

## 4. Kavita

**Método:** OIDC nativo desde la UI (disponible desde v0.8.8)
**Documentación oficial:** [wiki.kavitareader.com/guides/admin-settings/open-id-connect/](https://wiki.kavitareader.com/guides/admin-settings/open-id-connect/)

> ⚠️ **Requisito:** Versión 0.8.8 o superior.

### Paso 1 — Crear Application en Authentik

- **Client type:** Confidential
- **Redirect URIs:**
```
https://TU_KAVITA/signin-oidc
https://TU_KAVITA/signout-callback-oidc
```
- **Scopes:** `openid profile email`

### Paso 2 — Configurar OIDC en Kavita

Ir a **Settings (Admin) → Server Settings → OpenID Connect**

| Campo | Valor |
|---|---|
| Authority | `https://TU_AUTHENTIK/application/o/kavita/` |
| Client ID | Client ID de Authentik |
| Client Secret | Client Secret de Authentik |

Opciones adicionales recomendadas:

| Opción | Recomendación |
|---|---|
| Require Verified Email | ✅ Activado |
| Provision Accounts | ✅ Activado |
| Auto Login | Opcional |
| Disable Password Login | Solo si todos los usuarios usan SSO |

> ⚠️ Cualquier cambio en Authority o Client Secret requiere **reiniciar Kavita** manualmente.

### Paso 3 — Gestión de roles (opcional)

Crea un scope personalizado en Authentik con un claim `kavita-roles` con valores como `["Login", "Download"]` y configura en Kavita el campo **Roles Claim**.

---

## 5. Homebox

**Método:** OIDC nativo mediante variables de entorno
**Documentación oficial:** [homebox.software/en/quick-start/configure/oidc/](https://homebox.software/en/quick-start/configure/oidc/)

> ⚠️ Usa el fork activo **sysadminsmedia/homebox** (imagen: `ghcr.io/sysadminsmedia/homebox`). El repositorio original `hay-kot/homebox` está archivado.

### Paso 1 — Crear Application en Authentik

- **Client type:** Confidential
- **Redirect URI:**
```
https://TU_HOMEBOX/api/v1/users/login/oidc/callback
```
- **Scopes:** `openid profile email`

### Paso 2 — Configurar variables de entorno en Docker Compose

```yaml
environment:
  HBOX_OIDC_ENABLED: "true"
  HBOX_OIDC_ISSUER_URL: "https://TU_AUTHENTIK/application/o/homebox/.well-known/openid-configuration"
  HBOX_OIDC_CLIENT_ID: "TU_CLIENT_ID"
  HBOX_OIDC_CLIENT_SECRET: "TU_CLIENT_SECRET"
  HBOX_OPTIONS_TRUST_PROXY: "true"
  HBOX_OIDC_AUTO_REDIRECT: "true"
  HBOX_OIDC_VERIFY_EMAIL: "true"
  HBOX_OPTIONS_ALLOW_LOCAL_LOGIN: "false"    # Solo si quieres deshabilitar login local
```

---

## 6. Home Assistant

**Método:** OIDC mediante custom integration instalada por HACS
**Integration:** `hass-oidc-auth` ([github.com/christiaangoossens/hass-oidc-auth](https://github.com/christiaangoossens/hass-oidc-auth))
**Guía Authentik:** [integrations.goauthentik.io/miscellaneous/home-assistant/](https://integrations.goauthentik.io/miscellaneous/home-assistant/)

> ⚠️ No existe OIDC oficial en el core de Home Assistant. Esta integración es de la comunidad y está en estado **alpha**. Requiere HACS instalado.

### Paso 1 — Crear Application en Authentik

- **Client type:** Confidential
- **Redirect URI:**
```
https://TU_HOME_ASSISTANT/auth/oidc/callback
```
- **Scopes:** `openid profile email`

### Paso 2 — Instalar la integración via HACS

1. En Home Assistant → **HACS → Integrations**
2. Menú (⋮) → **Custom repositories**
3. Añadir: `https://github.com/christiaangoossens/hass-oidc-auth` → categoría **Integration**
4. Instalar "OIDC Auth"

### Paso 3 — Configurar en `configuration.yaml`

```yaml
oidc_auth:
  client_id: "TU_CLIENT_ID"
  client_secret: "TU_CLIENT_SECRET"
  discovery_url: "https://TU_AUTHENTIK/application/o/home-assistant/.well-known/openid-configuration"
```

### Paso 4 — Reiniciar y verificar

Tras reiniciar, accede a `https://TU_HOME_ASSISTANT/auth/oidc/welcome` para iniciar el flujo de login.

> 💡 Consulta el README del repositorio antes de cada actualización de HA para verificar la compatibilidad.

---

## 7. Navidrome

**Método:** Proxy Forward Auth + autenticación por cabecera HTTP
**Documentación Navidrome:** [navidrome.org/docs/getting-started/extauth-quickstart/](https://www.navidrome.org/docs/getting-started/extauth-quickstart/)

> ⚠️ Navidrome no tiene OIDC nativo. La solución protege la **web UI** pero los **clientes móviles deben seguir usando usuario/contraseña** de Navidrome. Deja el endpoint `/rest/` sin protección del proxy.

### Paso 1 — Crear Proxy Provider en Authentik

- **Mode:** Forward auth (single application)
- **External host:** `https://TU_NAVIDROME`

### Paso 2 — Configurar Navidrome para confiar en cabeceras

```yaml
environment:
  ND_REVERSEPROXYUSERHEADER: "X-authentik-username"
  ND_REVERSEPROXYWHITELIST: "172.16.0.0/12"
```

### Paso 3 — Configurar el reverse proxy (Nginx)

```nginx
location / {
    auth_request /outpost.goauthentik.io/auth/nginx;
    auth_request_set $authentik_username $upstream_http_x_authentik_username;
    proxy_set_header X-authentik-username $authentik_username;
    proxy_pass http://navidrome:4533;
}

location /rest/ {
    proxy_pass http://navidrome:4533/rest/;
}

location /share/ {
    proxy_pass http://navidrome:4533/share/;
}

location = /outpost.goauthentik.io/auth/nginx {
    internal;
    proxy_pass http://authentik-outpost:9000/outpost.goauthentik.io/auth/nginx;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
}
```

---

## 8. AdGuard Home

**Método:** Proxy Forward Auth
**Sin OIDC nativo** — [github.com/AdguardTeam/AdGuardHome/issues/1737](https://github.com/AdguardTeam/AdGuardHome/issues/1737)

> 💡 Una vez protegido con Authentik, puedes eliminar los usuarios internos de AdGuard delegando toda la autenticación al proxy.

### Paso 1 — Crear Proxy Provider en Authentik

- **Mode:** Forward auth (single application)
- **External host:** `https://TU_ADGUARD`

### Paso 2 — Configurar en el reverse proxy

```nginx
location / {
    auth_request /outpost.goauthentik.io/auth/nginx;
    proxy_pass http://adguardhome:3000;
}
```

---

## 9. metube

**Método:** Proxy Forward Auth
**Sin sistema de autenticación propio**

### Paso 1 — Crear Proxy Provider en Authentik

- **Mode:** Forward auth (single application)
- **External host:** `https://TU_METUBE`

### Paso 2 — Configurar en el reverse proxy

```nginx
location / {
    auth_request /outpost.goauthentik.io/auth/nginx;
    proxy_pass http://metube:8081;
}
```

---

## 10. TriliumNext

**Método:** Proxy Forward Auth (recomendado)
**Estado OIDC nativo:** ⚠️ Inestable con Authentik a abril 2026

> TriliumNext tiene OIDC implementado pero falla con Authentik porque espera claims en el ID token en lugar del endpoint `userinfo`.
> Issues: [#6444](https://github.com/TriliumNext/trilium/issues/6444) y [#6387](https://github.com/TriliumNext/Trilium/issues/6387).
> Usar Proxy Forward Auth hasta que se resuelvan los bugs.

### Opción A (Recomendada) — Proxy Forward Auth

Crear un Proxy Provider en Authentik y proteger el acceso mediante forward auth. Puerto por defecto: `:8080`.

### Opción B (Experimental) — OIDC nativo

```yaml
environment:
  TRILIUM_OAUTH_BASE_URL: "https://TU_AUTHENTIK/application/o/authorize/"
  TRILIUM_OAUTH_CLIENT_ID: "TU_CLIENT_ID"
  TRILIUM_OAUTH_CLIENT_SECRET: "TU_CLIENT_SECRET"
  TRILIUM_OAUTH_ISSUER_BASE_URL: "https://TU_AUTHENTIK/application/o/triliumnext/"
  TRILIUM_OAUTH_ISSUER_NAME: "Authentik"
```

---

## 11. MyTabs & Arcane

**Método:** Proxy Forward Auth
**Sin documentación de autenticación conocida**

Crear un Proxy Provider en Authentik y aplicar forward auth en el reverse proxy apuntando al puerto correspondiente de cada servicio.

---

## 📌 Resumen del plan de implementación

### Orden recomendado (de menor a mayor complejidad)

```
1.  Proxmox VE       → OIDC nativo, ~15 min
2.  Proxmox BS       → OIDC nativo, ~15 min (igual que PVE)
3.  Homebox          → OIDC nativo, variables de entorno
4.  Kavita           → OIDC nativo, UI admin (requiere v0.8.8+)
5.  Vaultwarden      → OIDC nativo, variables de entorno
6.  AdGuard / metube → Proxy Forward Auth (sencillo)
7.  MyTabs / Arcane  → Proxy Forward Auth
8.  Navidrome        → Proxy Forward Auth + config especial de cabecera
9.  TriliumNext      → Proxy Forward Auth (hasta que se resuelvan bugs)
10. Home Assistant   → OIDC vía HACS (alpha, más delicado)
```

### Prerequisitos globales

- [ ] Reverse proxy configurado (Nginx / Traefik / Caddy) con HTTPS para todos los servicios
- [ ] Authentik Outpost desplegado (para los servicios de Forward Auth)
- [ ] Todos los servicios accesibles solo a través del reverse proxy
- [ ] NTP sincronizado en todas las máquinas (crítico para OIDC)

---

## 📚 Referencias principales

- Authentik Integrations: [integrations.goauthentik.io](https://integrations.goauthentik.io)
- Authentik Proxy Provider: [goauthentik.io/docs/providers/proxy](https://goauthentik.io/docs/providers/proxy/)
- Vaultwarden SSO wiki: [github.com/dani-garcia/vaultwarden/wiki/Enabling-SSO-support-using-OpenId-Connect](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-SSO-support-using-OpenId-Connect)
- Kavita OIDC wiki: [wiki.kavitareader.com/guides/admin-settings/open-id-connect/](https://wiki.kavitareader.com/guides/admin-settings/open-id-connect/)
- Homebox OIDC docs: [homebox.software/en/quick-start/configure/oidc/](https://homebox.software/en/quick-start/configure/oidc/)