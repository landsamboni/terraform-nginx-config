# terraform-nginx-config

Infraestructura como código (IaC) con Terraform para desplegar y configurar un servidor **NGINX como reverse proxy** en un clúster **Amazon EKS**, como parte de la plataforma de viajes **Viajemos**.

---

## Descripción general

Este repositorio gestiona el despliegue de NGINX en Kubernetes (EKS) mediante Terraform. NGINX actúa como reverse proxy centralizado que enruta el tráfico hacia más de 30 microservicios internos, con soporte multi-país a través de subdominios por región.

**Stack tecnológico:**
- **Terraform** >= 1.8.0
- **AWS** (EKS, ALB, ACM, WAF v2)
- **Kubernetes** (namespace `bts-nginx`)
- **NGINX** (imagen `nginx:latest`)

---

## Arquitectura

```
Internet
   │
   ▼
AWS ALB (internet-facing)
   │  SSL/TLS termination (ACM)
   │  WAF v2 (viajemos-dev-waf-acl)
   ▼
Kubernetes Service (NodePort :80)
   │
   ▼
NGINX Deployment (bts-app-node-group)
   │  ConfigMap: nginx-config
   │  Mount: /etc/nginx/conf.d
   ▼
Microservicios internos (bts-app.svc.cluster.local)
   Puertos: 3000 (Node.js) | 6000 (otros servicios)
```

---

## Recursos Terraform desplegados

| Recurso | Nombre en clúster | Descripción |
|---|---|---|
| `kubernetes_config_map` | `nginx-config` | Almacena todos los archivos `.conf` de NGINX |
| `kubernetes_deployment` | `nginx-deployment` | Pod NGINX con 1 réplica |
| `kubernetes_service` | `nginx-deployment` | Servicio NodePort en puerto 80 |
| `kubernetes_ingress_v1` | `nginx-deployment` | Ingress ALB con SSL y WAF |

**Entorno AWS:**
- Región: `us-east-1`
- Clúster EKS: `viajemos-dev-eks-cluster`
- Node group: `bts-app-node-group`
- Namespace: `bts-nginx`

---

## Estructura del repositorio

```
terraform-nginx-config/
├── main.tf                        # Recursos principales de Kubernetes
├── providers.tf                   # Configuración de providers (AWS, Kubernetes)
├── variables.tf                   # Variables de entrada (región, nombre del clúster)
├── outputs.tf                     # Outputs (actualmente vacío)
└── nginx-config-files/            # Configuraciones NGINX
    ├── app.conf                   # App principal Viajemos
    ├── booking.conf               # Motor de reservas
    ├── cms.conf / cmsaws.conf     # CMS
    ├── crm.conf                   # CRM
    ├── customer.conf              # Portal de clientes
    ├── members.conf               # Membresías
    ├── leads.conf                 # Gestión de leads
    ├── searchengine.conf          # Motor de búsqueda general
    ├── searchenginehotels.conf    # Búsqueda de hoteles
    ├── app-miles.conf             # Programa de millas
    ├── hertzb2b.conf              # Integración Hertz B2B
    ├── policies.conf              # Gestión de políticas
    ├── rules.conf                 # Reglas de negocio
    ├── requests.conf              # Manejo de solicitudes
    ├── unfinish.conf              # Transacciones incompletas
    ├── app-vjs-argentina.conf     # ─┐
    ├── app-vjs-bolivia.conf       #  │
    ├── app-vjs-brasil.conf        #  │
    ├── app-vjs-canada.conf        #  │
    ├── app-vjs-chile.conf         #  │
    ├── app-vjs-colombia.conf      #  │  Instancias regionales
    ├── app-vjs-costa-rica.conf    #  │  (24 países)
    ├── app-vjs-ecuador.conf       #  │
    ├── app-vjs-espana.conf        #  │
    ├── app-vjs-mexico.conf        #  │
    ├── app-vjs-peru.conf          #  │
    ├── app-vjs-venezuela.conf     # ─┘
    └── ... (otros países)
```

---

## Configuración NGINX — patrón estándar

Cada archivo `.conf` sigue este patrón:

```nginx
server {
    listen 80;
    server_name <subdominio>.viajemosdev.info;

    # CORS — permite todos los orígenes
    # Métodos: OPTIONS, POST, GET

    location / {
        proxy_pass http://bts-app-<servicio>.bts-app.svc.cluster.local:<puerto>;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header PROJECT           viajemos;
        proxy_set_header SITE              viajemos_<pais>;  # solo instancias regionales
    }
}
```

**Multi-tenancy:** Los 24 dominios por país apuntan al mismo backend; la diferenciación se hace a través del header `SITE` (e.g., `viajemos_colombia`, `viajemos_argentina`).

---

## Prerequisitos

- Terraform >= 1.8.0
- AWS CLI configurado con acceso al clúster `viajemos-dev-eks-cluster`
- `kubectl` con kubeconfig apuntando al clúster
- Permisos sobre el namespace `bts-nginx`

---

## Uso

```bash
# Inicializar providers y módulos
terraform init

# Verificar el plan de cambios
terraform plan

# Aplicar los cambios
terraform apply
```

Para destruir los recursos:

```bash
terraform destroy
```

---

## Variables

| Variable | Descripción | Valor por defecto |
|---|---|---|
| `region` | Región de AWS | `us-east-1` |
| `cluster_name` | Nombre del clúster EKS | `viajemos-dev-eks-cluster` |

---

## Seguridad

- **SSL/TLS**: Terminación en el ALB con certificado ACM (`cae6da42-8ceb-4d0d-a41b-03ed24d4ddb3`), política `ELBSecurityPolicy-TLS13-1-2-2021-06`.
- **WAF**: Integración con `viajemos-dev-waf-acl` para protección de la capa de aplicación.
- **Red interna**: Todos los microservicios se resuelven vía DNS interno del clúster; no están expuestos directamente.

---

## Notas

- Los archivos de estado de Terraform (`*.tfstate`) están excluidos del repositorio vía `.gitignore`. Deben almacenarse en un backend remoto (S3 recomendado).
- La carpeta `Dominios Viajemos (enviados Yeyson) v2-v5/` contiene versiones históricas de configuraciones enviadas para revisión.
- El despliegue de Grafana fue removido; el monitoreo se gestiona a través de **DataDog**.
