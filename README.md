# Open edX en AWS con Terraform

Repositorio de despliegue de **Open edX** en **AWS** usando **Terraform**, con documentación
y tutorial completo en **español**.

> **Aviso importante:** Este proyecto se basa en
> [`open-craft/terraform-scripts`](https://github.com/open-craft/terraform-scripts),
> el repositorio oficial de scripts Terraform de OpenCraft para desplegar Open edX en AWS
> (licencia **GPL-3.0**, ver [`LICENSE`](./LICENSE)). Aquí se incluye una copia de sus
> módulos y el [tutorial interno de despliegue AWS](https://gitlab.com/opencraft/documentation/public/-/blob/master/tutorials/howtos/aws/AWS_terraform_deployment_tutorial.md)
> traducido al español y adaptado para usar los módulos de este mismo repositorio.

## Qué incluye

| Carpeta | Contenido |
|---|---|
| `modules/route53` | Zona hosted de Route53 + validación de certificados ACM |
| `modules/email` | Configuración de SES para que Open edX envíe correos |
| `modules/s3` | Buckets S3 (almacenamiento de cursos y tracking logs) + usuarios IAM |
| `modules/services/director` | Instancia "director" (bastión) para acceder al entorno |
| `modules/services/openedx` | Load balancers, security groups y registros Route53 de la instancia Open edX |
| `modules/services/sql` | Base de datos MySQL en Amazon RDS |
| `modules/services/elasticsearch` | Dominio de AWS Elasticsearch |
| `modules/services/mongodb` | Clúster MongoDB gestionado (opcional) |
| `modules/services/analytics` | Infraestructura para la instancia de Analytics (EMR, Elasticsearch, RDS) |
| `modules/services/cloudwatch` | Métricas de monitoreo y alertas SNS (prioridad alta/baja) |
| `modules/openedx` | Enfoque "legacy" (application, rds, s3, mongodb, wordpress) |
| `optional/auto_scaling` | Launch Template + Auto Scaling Group para los app servers |
| `optional/cloudfront_cdn` | CDN con CloudFront para cachear assets estáticos |

## Convenciones de nombrado

Todo recurso que provisione Terraform debe usar el siguiente esquema:

```
{prefix}-{environment}-{resource}-{additional_info}
```

- `prefix`: prefijo de nombre para todos los recursos provisionados (nombre del cliente).
- `environment`: `stage`, `production`, `upgrade`.
- `resource` y `additional_info`: los define cada módulo al instanciar los recursos.

Ejemplos:

- `opencraftx-production-appserver-1` — appserver del entorno de producción del cliente `OpenCraftX`
- `example-stage-alb` — load balancer del entorno stage del cliente `example`

Para facilitar la migración e importación, los módulos ofrecen variables de anulación de nombre
(`override_{resource_name}_name`) en aquellos recursos donde renombrar implicaría reemplazo o
cambio de credenciales.

### Categorías de módulos

- **Core**: provisiona una instancia Open edX "vanilla" y sus dependencias requeridas
  (buckets S3, bases de datos, load balancer, security groups, etc.). Viven en `modules/`.
- **Optional**: dependen de los recursos core y automatizan partes que cambian entre clientes
  (auto-escalamiento, CDN, etc.). Viven en `optional/`.

### Almacenamiento del estado

El archivo de estado de Terraform debe guardarse en un almacenamiento versionado y cifrado.
Opciones aceptables:

- [Backend S3 de AWS](https://www.terraform.io/docs/language/settings/backends/s3.html) (recomendado)
- GitLab Managed Terraform State

---

# Tutorial: desplegar Open edX en AWS con Terraform

Este tutorial te guía en el despliegue de edX en Amazon Web Services usando Terraform para los
recursos de AWS y Ansible para automatizar la instalación. Los módulos se toman de **este mismo
repositorio** (ruta relativa), aunque también puedes referenciar los originales de
[open-craft/terraform-scripts](https://github.com/open-craft/terraform-scripts) por git.

## Prerrequisitos

1. **Cuenta de AWS**: si es un cliente nuevo, conviene configurar un rol de asunción de rol
   (IAM Role) desde tu cuenta principal para poder acceder a la cuenta del cliente:
   - Tipo de entidad de confianza: *Another AWS account* (Otro cliente de AWS)
   - Account ID: el ID de la cuenta del cliente
   - Require external ID: False
   - Require MFA: True
   - Políticas adjuntas: `AdministratorAccess`
   - Nombre del rol: p. ej. `OpenCraft_Admin_Access`
2. **Cuenta de MongoDB Atlas** (https://www.mongodb.com/cloud): Open edX necesita un
   MongoDB; la opción recomendada es usar MongoDB Atlas.
3. **Región de AWS**: decide en qué región se alojarán los recursos.
4. **Delegación de dominio/subdominio**: decide qué dominio (o subdominio) apuntará a la
   instancia edX (el dominio raíz, donde vivirá el LMS). El propietario del dominio debe
   aceptar delegarlo a Route53 (se le entregarán los nameservers que genera el módulo
   `route53`, ver paso 1).
5. **Repositorio privado de configuración**: crea un repositorio (p. ej.
   `configuration-secure`) donde vivirán los scripts Terraform y Ansible y las
   configuraciones del cliente. Este repositorio debe ser **privado** porque contendrá
   credenciales.
6. **Dos pares de claves AWS**
   ([crear/importar pares de claves](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#how-to-generate-your-own-key-and-import-it-to-aws)):
   uno para acceder a la instancia *Director* y otro para las instancias edX.
7. **Terraform instalado** ([instalar Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)).

## Paso 1 — Proyecto de configuración única (Route53 + email)

Dentro del repositorio de configuración crea la siguiente estructura:

```
configuration-secure/
    terraform/
        setup/
        prod/
```

En `setup/main.tf`:

```hcl
provider "aws" {
  region = "<REGIÓN_AWS>"
}

module "route53" {
  source = "./../../modules/route53" # ajusta la ruta relativa a tu repositorio de módulos

  customer_domain = "<DOMINIO_DEL_CLIENTE>"
}

module "email" {
  source = "./../../modules/email"

  customer_domain        = "<DOMINIO_DEL_CLIENTE>"
  custom_emails_to_verify = ["mionombre@dominio.com"]  # correos de prueba (sandbox de SES)
  internal_emails        = ["no-reply@<DOMINIO_DEL_CLIENTE>"]
  route53_id             = module.route53.route53_id

  customer_name = "<NOMBRE_CLIENTE>"
  environment   = "prod"
}

output "route53_name_servers" {
  value = module.route53.route53_DNS
}
```

Notas:

- **Customer Domain**: el dominio raíz a configurar.
- **List of Emails**: correos de prueba que recibirán el mail mientras SES esté en sandbox.
- **List of Internal Emails**: correos que usará la instancia edX, normalmente
  `no-reply@<dominio>`.
- **Customer Environment**: pon `prod`.

Ejecuta:

```
terraform init
terraform apply
```

> **Nota:** en este punto es recomendable configurar un
> [backend de estado](https://www.terraform.io/docs/backends/types/s3.html) (S3, por ejemplo)
> para que el estado no quede solo guardado en tu computadora.

Terraform imprimirá los **Nameservers de Route53** que debes entregar al cliente para que haga
la delegación del dominio.

## Paso 2 — Proyecto de recursos diarios (prod)

Entra a `prod/` y crea `main.tf` (y guarda ahí también `director.pem`, la clave privada SSH del
director):

```
configuration-secure/
    terraform/
        setup/
            main.tf
        prod/
            main.tf
            director.pem
```

Contenido de `prod/main.tf` (cambia los valores entre `<...>`):

```hcl
provider "aws" {
  region = "<REGIÓN_AWS>"
}

module "director" {
  source = "./../../modules/services/director"

  director_key_pair_name = "<NOMBRE_KEY_PAIR_DIRECTOR>"
  image_id               = "<AMI_UBUNTU>"
  instance_type          = "t2.micro" # el más pequeño que encuentres
}

module "s3" {
  source = "./../../modules/s3"

  customer_name = "<NOMBRE_CLIENTE>"
  environment   = "prod"
}

module "openedx" {
  source = "./../../modules/services/openedx"

  customer_name              = "<NOMBRE_CLIENTE>"
  director_security_group_id = module.director.director_security_group_id
  environment                = "prod"
  customer_domain            = "<DOMINIO_DEL_CLIENTE>"
}

module "sql" {
  source = "./../../modules/services/sql"

  customer_name            = "<NOMBRE_CLIENTE>"
  environment              = "prod"

  allocated_storage        = 5
  database_root_username   = "opencraft"
  database_root_password   = "<GENERA_Y_GUARDA_UNA_CONTRASEÑA_SEGURA>"

  edxapp_security_group_id = module.openedx.edxapp_security_group_id

  instance_class           = "db.m3.medium" # o similar
}

module "elasticsearch" {
  source = "./../../modules/services/elasticsearch"

  customer_name            = "<NOMBRE_CLIENTE>"
  edxapp_security_group_id = module.openedx.edxapp_security_group_id
  environment              = "prod"
}

# Instancia edX (la creamos nosotros, el módulo openedx solo crea LB/SG/Route53)
resource "aws_instance" "edxapp" {
  ami                    = "<AMI_UBUNTU>"
  instance_type          = "t2.large" # o similar
  vpc_security_group_ids = [module.openedx.edxapp_security_group_id]

  root_block_device {
    volume_size = 50 # solo 50 GB para el disco de edxapp
    volume_type = "gp2"
  }

  key_name = "<NOMBRE_KEY_PAIR_EDX>"

  tags = {
    Name = "edxapp-1" # al añadir más instancias, cambia el número
  }
}

# Añade la instancia al load balancer de subdominios
resource "aws_lb_target_group_attachment" "edxapp" {
  target_group_arn = module.openedx.edxapp_lb_target_group_arn
  target_id        = aws_instance.edxapp.id
  port             = 80

  lifecycle {
    create_before_destroy = true
  }
}

# Añade la instancia al load balancer del dominio principal
resource "aws_lb_target_group_attachment" "edxapp_main_domain" {
  target_group_arn = module.openedx.edxapp_main_domain_lb_target_group_arn
  target_id        = aws_instance.edxapp.id
  port             = 80
}

# OUTPUTS

output "director_public_ip" {
  value = module.director.director_instance_public_ip
}

output "rds_host_name" {
  value = module.sql.mysql_host_name
}

output "elasticsearch_endpoint" {
  value = module.elasticsearch.elasticsearch
}

output "openedx_s3_storage_bucket_name" {
  value = module.s3.s3_storage_bucket_name
}

output "openedx_s3_storage_access_key" {
  value = module.s3.s3_storage_user_access_key
}

output "openedx_s3_storage_access_secret" {
  value = module.s3.s3_storage_user_secret_key
}

output "openedx_s3_tracking_logs_bucket_name" {
  value = module.s3.s3_tracking_logs_bucket_name
}

output "openedx_s3_tracking_logs_access_key" {
  value = module.s3.s3_tracking_logs_user_access_key
}

output "openedx_s3_tracking_logs_access_secret" {
  value = module.s3.s3_tracking_logs_user_secret_key
}
```

Valores a definir:

- **AWS Director Key Pair Name**: nombre del par de claves para SSH al director.
- **AWS Director Instance Type**: elige la instancia más pequeña.
- **SQL Allocated Storage**: normalmente `5` GB.
- **SQL Root Username**: normalmente `opencraft`.
- **SQL Root Password**: genera una contraseña segura y guárdala (la usarás después).
- **SQL Instance Class**: algo similar a `db.m3.medium`.
- **AWS edX Image ID**: un AMI de Ubuntu disponible en tu región.
- **AWS edX Instance Type**: algo similar a `t2.large`.
- **AWS edX Key Pair Name**: par de claves para SSH a la instancia edX desde el director.

Ejecuta:

```
terraform init
terraform apply
```

Terraform imprimirá los datos que usarás después para llenar tu archivo `vars.yml`
de Ansible.

## Paso 3 — Instalar Open edX (Ansible)

Con todos los recursos de AWS creados, sigue la
[guía oficial de OpenCraft para desplegar edX](https://gitlab.com/opencraft/documentation/public/-/blob/master/tutorials/howtos/aws/AWS_ELB_deployment_tutorial.md)
(desde la sección *MongoDB Atlas Database* en adelante) para continuar con la instalación de
edX usando Ansible y los datos de salida de Terraform.

### Sobre el módulo `openedx` (importante)

El módulo `modules/services/openedx` **no crea** la instancia EC2 de Open edX: prepara todo lo
que la instancia necesita (2 load balancers — uno para subdominios y otro para el dominio raíz,
con sus security groups y listeners —, security group de la instancia y registros Route53).
Tú debes crear la instancia (con `user_data` que instale lo necesario, ver
[`setup_edx_instance.sh.example`](./modules/services/openedx/setup_edx_instance.sh.example))
y añadirla a los dos target groups, como se muestra en el Paso 2. Esto se hace cada vez que se
despliega una instancia nueva (release, fix, etc.).

---

# Referencia de módulos (resumen en español)

## modules/route53 — AWS Route53 Setup

Placeholder sencillo para inicializar Route53, usado por los módulos `email` y `services/openedx`.

**Supuestos**

- El cliente acepta delegar el control completo del dominio/subdominio (deberá configurar sus
  NS hacia los nameservers de Route53).

**Entradas**

- `customer_domain` (requerido): dominio o subdominio delegado a Route53.
- `customer_domain_extra_records` (opcional): mapa de registros DNS adicionales; la clave es el
  `name` del registro y `value`, `type`, `ttl` se toman del objeto.
- `enable_acm_validation` (opcional, por defecto `true`): ponlo en `false` si los registros aún
  no van a validar.

**Salidas**

- `route53_DNS`: nameservers (entregar al cliente).
- `route53_id`: para usar en otros módulos.

## modules/email — AWS Emails Setup

Configura SES para que Open edX pueda enviar correos (y algunos de prueba/recepción).

**Entradas**

- `customer_domain`: dominio raíz delegado a Route53.
- `route53_id`: ID de la zona Route53 configurada.
- `internal_emails`: lista de correos que usará la instancia (`no-reply` se añade por defecto).
- `custom_emails_to_verify`: correos de prueba como destinatarios mientras SES está en sandbox.
- `customer_name`
- `environment`

*Nota:* después de crearlo, entra a la consola de AWS → SES y solicita salir del sandbox para
poder enviar correos a cualquier destinatario verificado o no.

## modules/s3 — AWS S3 Setup

Contiene los buckets S3 necesarios para que Open edX funcione: uno para almacenamiento de
cursos y otro para tracking logs.

**Entradas** (solo personalización)

- `customer_name`
- `environment`: p. ej. `prod`

**Salidas** (la mayoría necesarias para la instalación/configuración de Open edX; irán al
archivo `vars.yml` de Ansible)

- `s3_storage_bucket_name` → `EDXAPP_AWS_STORAGE_BUCKET_NAME`
- `s3_storage_user_access_key` → `AWS_ACCESS_KEY_ID`
- `s3_storage_user_secret_key` → `AWS_SECRET_ACCESS_KEY`
- `s3_tracking_logs_bucket_name` → `COMMON_OBJECT_STORE_LOG_SYNC_BUCKET`
- `s3_tracking_logs_user_access_key` → `AWS_S3_LOGS_ACCESS_KEY_ID`
- `s3_tracking_logs_user_secret_key` → `AWS_S3_LOGS_SECRET_KEY`

## modules/services/director — The Director Instance

Instancia secundaria (bastión) usada para la instalación de Open edX. El acceso a la instancia
edX se hace **siempre a través del director**; la instancia edX no es accesible por SSH desde
Internet, solo el director puede acceder a ella.

**Supuestos**

- Ya creaste/importaste un *Key Pair* en AWS (por la consola) y tienes las claves pública y
  privada. Deberías crear **dos** (una para el director y otra para las instancias edX).

**Entradas** (obtén los valores de la consola de AWS para tu región)

- `image_id`: AMI de la instancia (p. ej. `ami-0edab4XXXXXXXXX`).
- `instance_type`: tipo de instancia; se recomienda el más pequeño (p. ej. `t2.micro`).
- `director_key_pair_name`: nombre del Key Pair para SSH al director.
- `ebs_volume_type`: tipo de volúmenes EBS (por defecto `gp2`).
- `ebs_volume_size`: tamaño de los volúmenes EBS en GiB (por defecto 8).
- `ebs_iops`: IOPS base, solo aplica si `ebs_volume_type` es `gp3` (por defecto 3000).

**Salidas**

- `director_public_ip`: IP pública del director (informativa, para saber a cuál hacer SSH).

Para SSH: `ssh -i director.pem <usuario>@<director_public_ip>` (en instancias Ubuntu el usuario
es `ubuntu`).

## modules/services/openedx — Instancia Open edX (LB, SG y DNS)

Prepara todo lo que la instancia edX necesita para "enchufarla" en tu propio script Terraform
(ver Paso 2 del tutorial). **No crea** la instancia EC2.

**Supuestos**

- Existe una zona hosted de Route53 y sus certificados ACM emitidos (ver módulo `route53`).

**Recursos que crea**

- 2 **Application Load Balancers** (uno para subdominios, otro para el dominio principal) con
  sus security groups y listeners.
- Un **Security Group** para la instancia edX.
- Los registros **Route53** necesarios para los subdominios edX.

**Entradas principales**

- `customer_domain`: debe coincidir con el dominio configurado en Route53.
- `customer_name`
- `environment`
- `director_security_group_id`: SG del director (ver módulo `director`).
- `route53_subdomains`: subdominios a crear como registros apuntando al LB.
- `lb_idle_timeout`: idle timeout del load balancer.
- `enable_https`: `false` si el ACM aún no está emitido o no necesitas https.
- `lb_ssl_security_policy`: política SSL de AWS (por defecto `ELBSecurityPolicy-2016-08`).
- `enable_lb_stickiness`
- `lb_stickiness_duration`

## modules/services/sql — AWS RDS (MySQL)

Open edX necesita MySQL; este módulo configura RDS accesible **solo** desde un security group
de AWS.

**Entradas principales**

- `customer_name`, `environment`
- `instance_class`: clase de la instancia RDS (p. ej. `db.m3.medium`).
- `allocated_storage`: almacenamiento en GB.
- `database_root_username`: usuario root (normalmente `opencraft`) → `EDXAPP_MYSQL_USER`.
- `database_root_password` → `EDXAPP_MYSQL_PASSWORD`.
- `edxapp_security_group_id`: SG de la instancia edX.
- `max_allocated_storage`: por defecto 100 (activa auto-escalamiento; 0 para desactivar).
- `extra_security_group_ids`: SG adicionales para el RDS principal.
- `number_of_replicas`: réplicas (por defecto 0).
- `replica_extra_security_group_ids`, `replica_publicly_accessible` (def. `false`),
  `enable_replica_multi_az` (def. `false`).

**Salidas**

- `mysql_host_name`: endpoint → `EDXAPP_MYSQL_HOST`.
- `mysql_instance_id`: ID (para configurar alarmas CloudWatch).

## modules/services/elasticsearch — AWS Elasticsearch

Dominio de Elasticsearch con acceso completo, pero **solo** desde el security group indicado
(el de la instancia edX).

**Entradas principales**

- `customer_name`, `environment`
- `edxapp_security_group_id`: SG de la instancia edX.
- `elasticsearch_version` (def. `1.5`)
- `elasticsearch_instance_type` (def. `t2.small.elasticsearch`)
- `zone_awareness_enabled` (def. `true`), `availability_zone_count` (def. 2)
- `dedicated_master_enabled` (def. `true`)
- `extra_security_group_ids`, `instance_count` (def. 2)
- `specific_subnet_ids`, `specific_vpc_id`, `specific_domain_name`
- `ebs_volume_type` (def. `gp2`), `ebs_volume_size` (def. 10 GiB), `ebs_iops` (def. 3000)

**Salidas**

- `elasticsearch`: endpoint → `ELASTICSEARCH_HOST` (sin la parte `https`).

## modules/services/mongodb — Clúster MongoDB

Clúster MongoDB personalizado mantenido en AWS.

> **IMPORTANTE**: esto no está incluido en los planes de mantenimiento estándar; se recomienda
> empujar al cliente a usar **MongoDB Atlas** en su lugar.

**Entradas**

- `number_of_instances`, `image_id`, `instance_type`, `customer_name`, `environment`
- `edxapp_security_group_id`: SG para acceder (el de las instancias edX).
- `openedx_key_pair_name`: Key Pair para SSH a estas instancias.

**Salidas**

- `mongodb_instances`: IPs privadas (sirven para `EDXAPP_MONGO_HOSTS`).

*Nota*: aun así debes configurar un MongoDB productivo dentro de las instancias (ReplicaSet,
auth y SSL).

## modules/services/analytics — Recursos de Analytics

Recursos AWS necesarios para la instancia de Analytics (ver
[docs de analytics](https://github.com/open-craft/openedx-deployment/blob/master/docs/analytics/AWS_setup.md)).

**Supuestos**: ya tienes una instancia edX desplegada en AWS.

**Entradas principales**

- `customer_name`, `environment` (normalmente `analytics`)
- `analytics_image_id`, `analytics_instance_type`, `analytics_key_pair_name`
- `hosted_zone_domain`: zona donde se crea el subdominio de insights (normalmente igual al
  dominio del LMS)
- `instance_iteration`: número de iteración de la instancia (empieza en 1)
- Volúmenes EBS: `instance_ebs_volume_type/size/iops`
- Elasticsearch: `elasticsearch_instance_type`, `elasticsearch_ebs_volume_type/size/iops`,
  `elasticsearch_version`
- `edxapp_rds_port` (def. 3306), `director_security_group_id`
- Buckets de grades: `edxapp_s3_grade_bucket_id/arn`, `edxapp_s3_grade_user_arn`
- `number_of_instances` (def. 1), `lb_instance_indexes` (def. `[0]`)
- `use_route53` (def. `true`), `aws_vpc_id` (vacío = VPC por defecto)

**Salidas**

- `analytics_security_group_id`
- `emr_rds_security_group_id`

## modules/services/cloudwatch — Monitoreo y alertas

Configura métricas estándar de monitoreo y dos topics SNS: uno para alertas de prioridad baja y
otro de prioridad alta.

Ejemplo de configuración:

```hcl
module "cloud_watch" {
  source                  = "../modules/services/cloudwatch"
  default_ec2_alarm_enabled = true
  ec2_instances           = [aws_instance.app_server.id]
  default_rds_alarm_enabled = true
  rds_instances           = [aws_db_instance.rds.id]
  environment             = "prod"
  customer_name           = "cliente"
}

# Suscripción de email a alertas de baja prioridad
resource "aws_sns_topic_subscription" "low_priority_email" {
  topic_arn = module.cloud_watch.low_priority_alert_arn
  protocol  = "email"
  endpoint  = "baja.prioridad@ejemplo.com"
}
```

**Entradas**: `customer_name`, `environment`, `default_ec2_alarms_enabled` (def. `false`),
`ec2_instances`, `default_rds_alarms_enabled` (def. `false`), `rds_instances`.

**Salidas**: `low_priority_alert_arn`, `high_priority_alert_arn`,
`server_logs_profile_name` (profile de instancia para subir eventos a CloudWatch).

## optional/auto_scaling — Auto Scaling Group

Crea un Launch Template y un grupo de auto-escalamiento y lo conecta al load balancer existente.
Crea AMIs a partir de las instancias "fuente" que le pases (las instancias se detienen durante
el snapshot, hay un período de downtime).

**Entradas principales**: `aws_vpc_id`, `client_shortname`, `environment`, `instances`
(lista de instancias fuente), `lb_target_group_arn`, `security_group_id`, `release` (nombre del
release de Open edX), y opciones de scaling (`min/max/desired`, métricas, tipo de instancia).

## optional/cloudfront_cdn — CDN para assets estáticos

CDN de CloudFront para cachear assets de una URL de origen (instancia Open edX o sitio de
marketing).

1. Añade al TF de tu cliente:

```hcl
module "openedx_cdn" {
  source         = "../optional/cloudfront_cdn"
  client_shortname = "cliente"
  environment      = "prod"
  service_name     = "cdn-edxapp"
  origin_domain    = "cursos.cliente.test"
}
```

2. Publica la salida:

```hcl
output "openedx_cdn_domain_name" {
  value = module.openedx_cdn.aws_cloudfront_distribution.domain_name
}
```

3. Tras `terraform apply`, añade el valor a `vars.yml`:
   `EDXAPP_LMS_STATIC_URL_BASE` = `openedx_cdn_domain_name`.

Opcional: dominio personalizado del CDN (`aliases`, `alias_zone_id`, `alias_name`,
`alias_certificate_arn` — el certificado ACM debe estar en `us-east-1`).

---

# Licencia y atribución

- Los módulos Terraform provienen de
  [open-craft/terraform-scripts](https://github.com/open-craft/terraform-scripts) © OpenCraft,
  bajo licencia **GPL-3.0** (ver [`LICENSE`](./LICENSE)).
- El tutorial original en inglés está en
  [open-craft/documentation (GitLab)](https://gitlab.com/opencraft/documentation/public/-/blob/master/tutorials/howtos/aws/AWS_terraform_deployment_tutorial.md).
- Traducción al español y adaptación: este repositorio.
