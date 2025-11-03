# Cloud Config Server.

##  Descripción

Cloud Config Server es un servicio de **Spring Cloud Config** que centraliza la configuración de todos los microservicios desde un repositorio Git. Permite gestionar configuraciones por ambiente (dev, stage, prod) sin necesidad de recompilar aplicaciones.

##  Propósito

- **Configuración Centralizada**: Todas las configuraciones en un solo lugar (repositorio Git)
- **Gestión por Ambiente**: Diferentes configuraciones para dev/stage/prod
- **Actualización sin Reiniciar**: Los servicios pueden refrescar configuración sin reiniciar
- **Versionado**: Historial completo de cambios de configuración

##  Arquitectura

```
┌──────────────────┐
│  Cloud Config    │ ──lee──> ┌──────────────────┐
│     Server       │          │  Repositorio Git  │
│   Puerto: 9296   │          │  (config files)   │
└──────────────────┘          └──────────────────┘
         ▲
         │ consulta configuración
         │
┌────────┴────────┐
│  Microservicios │
│ (Product, Order,│
│    User, etc)   │
└─────────────────┘
```

##  Configuración

### Puerto
- **Puerto**: `9296`
- **URL Local**: `http://localhost:9296`
- **URL Kubernetes**: `http://cloud-config.ecommerce-dev.svc.cluster.local:9296`

### Repositorio Git
```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/SelimHorri/cloud-config-server
          clone-on-start: true
```

### Application Name
- **Nombre**: `CLOUD-CONFIG`

##  Endpoints

### Health Check
```
GET /actuator/health
```

### Configuración de un Servicio
```
GET /{application}/{profile}
GET /{application}/{profile}/{label}
```

Ejemplos:
- `GET /product-service/dev` - Configuración de product-service para dev
- `GET /order-service/stage` - Configuración de order-service para stage

### Buscar Configuración
```
GET /{application}/{profile}/{label}/{path}
```

##  Integración con Microservicios

Los microservicios se conectan a Cloud Config usando:

```yaml
spring:
  config:
    import: optional:configserver:http://cloud-config.ecommerce-dev.svc.cluster.local:9296
```

**Nota**: `optional:` significa que si Cloud Config no está disponible, el servicio seguirá funcionando con su configuración local.

##  Despliegue

### Desarrollo Local

```bash
./mvnw spring-boot:run
```

Servicio disponible en: `http://localhost:9296`

### Docker

```bash
docker build -t cloud-config:0.1.0 .
docker run -p 9296:9296 cloud-config:0.1.0
```

### Kubernetes

El servicio se despliega automáticamente mediante el pipeline CI/CD en el namespace `ecommerce-dev`.

##  Estructura del Repositorio de Configuración

El repositorio Git debe tener esta estructura:

```
cloud-config-server/
├── product-service/
│   ├── application-dev.yml
│   ├── application-stage.yml
│   └── application-prod.yml
├── order-service/
│   ├── application-dev.yml
│   ├── application-stage.yml
│   └── application-prod.yml
└── user-service/
    ├── application-dev.yml
    ├── application-stage.yml
    └── application-prod.yml
```

##  Flujo de Configuración

1. **Microservicio inicia** → Solicita configuración a Cloud Config
2. **Cloud Config** → Lee archivo del repositorio Git según `{application}-{profile}.yml`
3. **Cloud Config** → Retorna configuración al microservicio
4. **Microservicio** → Usa la configuración recibida

### Refresh Manual (Sin Reiniciar)

Los microservicios pueden refrescar su configuración sin reiniciar usando:

```
POST /actuator/refresh
```

## Notas Importantes

### Característica "Optional"

La configuración usa `optional:configserver`, lo que significa:
-  Si Cloud Config está disponible → usa configuración del Git
-  Si Cloud Config NO está disponible → usa `application.yml` local
-  El servicio nunca falla por falta de Cloud Config

### Estrategia de Despliegue

- **Namespace**: Siempre `ecommerce-dev` (mismo para dev/stage/prod)
- **Tags de Imagen**:
  - `dev-latest` (branches dev/develop)
  - `stage-latest` (branch stage)
  - `prod-0.1.0` (branches main/master)
- **Replicas**: 1 (servicio singleton)

### Orden de Arranque

Cloud Config puede iniciar en cualquier momento ya que los servicios usan `optional:`. Sin embargo, es recomendable desplegarlo después de Service Discovery:

1. Service Discovery
2. **Cloud Config** (opcional pero recomendado)
3. Microservicios de negocio
4. API Gateway
5. Proxy Client

##  Testing

Este servicio no requiere pruebas unitarias o de integración ya que:
- Es un servicio estándar de Spring Cloud Config
- No tiene lógica de negocio personalizada
- Solo necesita estar desplegado y funcionando

