# Red de la biblioteca · L9

Memoria medida con `docker stats --no-stream` en el equipo, con los 17 contenedores arriba y la noche
del simulador ya consumida.

## AWS · VPC 10.0.0.0/16 · us-east-1

| Subred | Rango | Zona | Ruta 0.0.0.0/0 |
|---|---|---|---|
| pública | 10.0.1.0/24 | us-east-1a | Internet Gateway |
| aplicación | 10.0.11.0/24 | us-east-1a | interfaz de red de `nat` |
| datos-a | 10.0.21.0/24 | us-east-1a | interfaz de red de `nat` |
| datos-b | 10.0.22.0/24 | us-east-1b | interfaz de red de `nat` |

| Máquina | Subred | IP privada | Tamaño | Lo que corre | Por qué ese tamaño |
|---|---|---|---|---|---|
| `gw-aws` | pública | 10.0.1.20, con Elastic IP | t3.small | Caddy con HTTPS y `biblioteca-gateway` | El gateway mide 46 MiB; con Caddy y el sistema queda muy por debajo de 2 GiB, y una t3.nano (0,5 GiB) quedaria justa para el sistema, Docker y la renovacion de certificados |
| `nat` | pública | 10.0.1.10 | t3.nano | NAT instance de las subredes privadas | No corre contenedores: solo reenvia paquetes de las subredes privadas, y para eso 0,5 GiB sobran |
| `servicios` | aplicación | 10.0.11.10 | t3.medium | `biblioteca-libros`, `biblioteca-prestamos` y `biblioteca-eventos` | Suman 213 MiB (21,7 + 91,8 + 99,6); los 4 GiB dejan margen para los picos del consumidor de Kafka al reprocesar y para crecer con mas consumidores |
| `gestion` | aplicación | 10.0.11.20 | t3.medium | `biblioteca-admin`, Kafka UI, `biblioteca-simulador`, runner `aws` (Prometheus y Grafana, opcionales) | Admin 61 MiB y Kafka UI 411 MiB suman 472 MiB; con el simulador, el runner y Prometheus y Grafana no cabe holgado en 2 GiB, por eso 4 GiB |
| `mq-1` | datos-a | 10.0.21.11 | t3.small | controller 1, broker 4 y `rabbit1` | Suman 800 MiB (179 + 482 + 139); caben en 2 GiB con margen para el cache de pagina que usa Kafka |
| `mq-2` | datos-b | 10.0.22.12 | t3.small | controller 2, broker 5 y `rabbit2` | Suman 653 MiB (162 + 355 + 136); caben en 2 GiB con margen |
| `mq-3` | datos-a | 10.0.21.13 | t3.small | controller 3 y broker 6 | Suman 515 MiB (163 + 352); caben en 2 GiB con margen |
| `datos` | datos-a | 10.0.21.30 | t3.small | Postgres 18 | Postgres mide 44 MiB hoy; los 2 GiB dejan espacio para el cache de la base y su crecimiento |

| Grupo de reglas | Puerto | Origen | Para qué |
|---|---|---|---|
| `sg-gw` | 80 | `0.0.0.0/0` | Desafío de Let's Encrypt y redirección a HTTPS |
| `sg-gw` | 443 | `pendiente E2` (IP pública de gw-azure)/32 | La API, solo para el BFF |
| `sg-servicios` | 3001, 3002 y 3010 | `sg-gw` y `sg-gestion` | El gateway llega a `libros` y `prestamos`; el administrador revisa `eventos` |
| `sg-mq` | 9092 | `sg-servicios`, `sg-gestion` y `sg-mq` | Clientes de Kafka y réplica entre brokers |
| `sg-mq` | 9093 | `sg-mq` | Quórum KRaft |
| `sg-mq` | 5672 | `sg-servicios` y `sg-gestion` | AMQP de RabbitMQ |
| `sg-mq` | 15672 | `sg-gestion` | Administración de RabbitMQ |
| `sg-mq` | 4369 y 25672 | `sg-mq` | Clúster de RabbitMQ |
| `sg-datos` | 5432 | `sg-servicios` y `sg-gestion` | Postgres |
| `sg-nat` | todos | `sg-servicios`, `sg-gestion`, `sg-mq` y `sg-datos` | Salida a internet de las subredes privadas |
| `sg-gestion` | ninguno | | Se entra por Session Manager |

## Azure · VNet 10.1.0.0/16 · `brazilsouth`

Nota: en esta suscripcion `brazilsouth` es la unica region que acepta recursos, pero ahi `Standard_B2ats_v2`
y `Standard_B2als_v2` aparecen como NotAvailableForSubscription (ver `deploy/azure/region.txt`). Se aviso al
docente; los tamanos de abajo son los del diseno.

| Subred | Rango | Salida a internet |
|---|---|---|
| pública | 10.1.1.0/24 | directa, por la IP pública de `gw-azure` |
| privada | 10.1.2.0/24 | por `gw-azure` (tabla de rutas hacia 10.1.1.4); creada con `--default-outbound false` |

| Máquina | Subred | IP privada | Tamaño | Lo que corre | Por qué ese tamaño |
|---|---|---|---|---|---|
| `gw-azure` | pública | 10.1.1.4, con IP pública estática y etiqueta `pendiente E2` | B2ats_v2 | Caddy, `biblioteca-web`, `biblioteca-gateway` y NAT de la subred privada | Web 11 MiB y gateway 46 MiB suman 57 MiB; con Caddy y el NAT caben en 1 GiB |
| `bff` | privada | 10.1.2.4, sin IP pública | B2als_v2 | `biblioteca-bff` y el runner `azure` | El BFF mide 45 MiB; los 4 GiB son para el runner, que compila las imagenes al desplegar |

| NSG | Puerto | Origen | Para qué |
|---|---|---|---|
| `gw-azure` | 443 y 80 | Internet | La web y el desafío de Let's Encrypt |
| `gw-azure` | 22 | `pendiente E2` (IP de cada integrante)/32 | SSH |
| `gw-azure` | todos | 10.1.2.0/24 | El NAT de `bff` |
| `bff` | todos | 10.1.0.0/16 | Solo desde la VNet |
