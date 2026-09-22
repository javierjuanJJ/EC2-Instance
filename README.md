# Proyecto: Sitio Estático en AWS EC2

> **Objetivo:** Crear una instancia Amazon EC2 con Ubuntu Server, conectarte por SSH, instalar Nginx y desplegar un sitio web estático accesible desde internet por su IP pública.

---

## Índice

1. [Descripción del proyecto](#1-descripción-del-proyecto)
2. [Requisitos previos](#2-requisitos-previos)
3. [Configuración de AWS CLI](#3-configuración-de-aws-cli)
4. [Obtener la AMI de Ubuntu](#4-obtener-la-ami-de-ubuntu)
5. [Crear el par de claves (key pair)](#5-crear-el-par-de-claves-key-pair)
6. [Crear el grupo de seguridad](#6-crear-el-grupo-de-seguridad)
7. [Lanzar la instancia EC2](#7-lanzar-la-instancia-ec2)
8. [Conectar por SSH](#8-conectar-por-ssh)
9. [Instalar y configurar Nginx](#9-instalar-y-configurar-nginx)
10. [Desplegar el sitio web estático](#10-desplegar-el-sitio-web-estático)
11. [Verificar el sitio en el navegador](#11-verificar-el-sitio-en-el-navegador)
12. [Hacer lo mismo en Floci (emulador local de AWS)](#12-hacer-lo-mismo-en-floci-emulador-local-de-aws)
13. [Stretch goals](#13-stretch-goals)
14. [Limpieza de recursos](#14-limpieza-de-recursos)
15. [Resultados de aprendizaje](#15-resultados-de-aprendizaje)
16. [Solución de problemas](#16-solución-de-problemas)

---

## 1. Descripción del proyecto

Este proyecto cubre el ciclo completo de una infraestructura mínima en la nube:

| # | Tarea | Herramienta |
|---|-------|-------------|
| 1 | Crear cuenta AWS | AWS Console |
| 2 | Lanzar instancia Ubuntu Server `t2.micro` (Free Tier) | AWS CLI |
| 3 | Configurar grupo de seguridad (puertos 22 y 80) | AWS CLI |
| 4 | Conectarse por SSH con par de claves | OpenSSH |
| 5 | Actualizar el sistema e instalar Nginx | SSH / apt |
| 6 | Desplegar un sitio HTML estático | scp / SSH |
| 7 | Acceder al sitio por la IP pública | Navegador |

**Especificaciones de la instancia:**

- **AMI:** Ubuntu Server 22.04 LTS (Canonical)
- **Tipo:** `t2.micro` (elegible para AWS Free Tier)
- **VPC / Subred:** VPC y subred por defecto de la región
- **IP pública:** Asociada al lanzamiento
- **Grupo de seguridad:** Inbound TCP 22 (SSH) y TCP 80 (HTTP) desde `0.0.0.0/0`
- **Par de claves:** Creado en este tutorial

---

## 2. Requisitos previos

- Una [cuenta de AWS](https://aws.amazon.com/free/) (si no tienes, créala en la consola; te pedirán tarjeta de crédito, pero `t2.micro` es gratis dentro del Free Tier)
- AWS CLI v2 instalado en tu máquina local (macOS, Linux o WSL)
- Un cliente SSH (`ssh` viene incluido en la mayoría de sistemas)
- Acceso a una terminal

Instala AWS CLI según tu sistema:

**macOS:**

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

**Linux (x86_64):**

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Windows:** descarga el [instalador oficial](https://aws.cli.amazoncli.com/latest/AWSCLIV2.msi).

Verifica la instalación:

```bash
aws --version
# aws-cli/2.x.x Python/3.x.x ...
```

---

## 3. Configuración de AWS CLI

Configura tus credenciales de IAM (Consola AWS → *My Security Credentials* → *Create access key*):

```bash
aws configure
```

Te pedirá:

```
AWS Access Key ID [None]: <TU_ACCESS_KEY>
AWS Secret Access Key [None]: <TU_SECRET_KEY>
Default region name [None]: us-east-1
Default output format [None]: json
```

Verifica que funciona:

```bash
aws sts get-caller-identity
```

Debes ver un JSON con tu `Account`, `Arn` y `UserId`.

Define una variable de región para usarla en todos los pasos:

```bash
export AWS_REGION="us-east-1"
```

---

## 4. Obtener la AMI de Ubuntu

En lugar de hardcodear un ID de AMI (que caduca), obtén la última AMI oficial de Ubuntu 22.04 LTS publicada por Canonical (`Owner 099720109477`):

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

echo "AMI ID: $AMI_ID"
```

> **Nota:** Este es el mismo patrón que usa la [documentación oficial de AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/bash_ec2_code_examples.md) para resolver dinámicamente el ID de una AMI.

---

## 5. Crear el par de claves (key pair)

Crea un par de claves y guarda el archivo privado `.pem` localmente:

```bash
KEY_NAME="ec2-static-site-key"

aws ec2 create-key-pair \
  --key-name "$KEY_NAME" \
  --query 'KeyMaterial' \
  --output text > "${KEY_NAME}.pem"

chmod 400 "${KEY_NAME}.pem"
```

> **Importante:** SSH rechaza llaves con permisos abiertos. `chmod 400` es obligatorio; sin él la conexión fallará con el error *"UNPROTECTED PRIVATE KEY FILE"*.

Si ya tienes un par de claves, omite este paso y usa su nombre en los comandos siguientes.

---

## 6. Crear el grupo de seguridad

### 6.1. Obtener el VPC por defecto

```bash
DEFAULT_VPC=$(aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

echo "Default VPC: $DEFAULT_VPC"
```

### 6.2. Crear el grupo de seguridad

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name "static-site-sg" \
  --description "SSH (22) y HTTP (80) para sitio estatico" \
  --vpc-id "$DEFAULT_VPC" \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SG_ID"
```

### 6.3. Permitir SSH (puerto 22)

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "0.0.0.0/0"
```

> **Recomendación de seguridad:** en producción, restringe el acceso SSH a tu IP pública actual en lugar de `0.0.0.0/0`:
>
> ```bash
> MI_IP=$(curl -s http://checkip.amazonaws.com)/32
> aws ec2 authorize-security-group-ingress \
>   --group-id "$SG_ID" \
>   --protocol tcp --port 22 --cidr "$MI_IP"
> ```

### 6.4. Permitir HTTP (puerto 80)

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "0.0.0.0/0"
```

### 6.5. Verificar las reglas

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --query 'SecurityGroups[0].IpPermissions'
```

---

## 7. Lanzar la instancia EC2

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --count 1 \
  --instance-type t2.micro \
  --key-name "$KEY_NAME" \
  --security-group-ids "$SG_ID" \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=static-site}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance ID: $INSTANCE_ID"
```

| Parámetro | Valor | Propósito |
|-----------|-------|-----------|
| `--image-id` | `$AMI_ID` | Ubuntu Server 22.04 LTS |
| `--instance-type` | `t2.micro` | Free Tier (1 vCPU, 1 GB RAM) |
| `--key-name` | `$KEY_NAME` | Acceso SSH |
| `--security-group-ids` | `$SG_ID` | Puertos 22 y 80 abiertos |
| `--associate-public-ip-address` | — | IP pública para SSH y HTTP |

### 7.1. Esperar a que esté en estado `running`

```bash
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
echo "La instancia está en estado running"
```

### 7.2. Obtener la IP pública

```bash
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "IP pública: $PUBLIC_IP"
```

---

## 8. Conectar por SSH

El usuario por defecto de las AMIs de Ubuntu en EC2 es **`ubuntu`** (documentado en la [guía de conexión de EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.md)):

```bash
ssh -i "${KEY_NAME}.pem" "ubuntu@${PUBLIC_IP}"
```

Si es la primera conexión, acepta la huella del host escribiendo `yes`.

Una vez dentro, actualiza el sistema:

```bash
sudo apt update && sudo apt upgrade -y
```

Para salir del servidor: `exit`.

---

## 9. Instalar y configurar Nginx

Conéctate de nuevo si saliste y ejecuta:

```bash
ssh -i "${KEY_NAME}.pem" "ubuntu@${PUBLIC_IP}"
```

Dentro del servidor:

```bash
sudo apt update
sudo apt install -y nginx
```

Activa e inicia Nginx:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

Verifica localmente en el servidor:

```bash
curl -I http://localhost
# HTTP/1.1 200 OK
```

---

## 10. Desplegar el sitio web estático

### 10.1. Crear el archivo `index.html` localmente

En tu máquina local (fuera del servidor):

```bash
mkdir -p sitio-estatico && cat > sitio-estatico/index.html << 'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Mi Sitio en AWS EC2</title>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 640px; margin: 4rem auto; padding: 0 1rem; color: #232f3e; }
    h1 { color: #ff9900; }
    .badge { display: inline-block; background: #232f3e; color: #fff; padding: .25rem .75rem; border-radius: 999px; font-size: .85rem; }
  </style>
</head>
<body>
  <h1>¡Hola desde AWS EC2! 🚀</h1>
  <p>Sitio estático desplegado con <span class="badge">Ubuntu 22.04</span> <span class="badge">Nginx</span> <span class="badge">t2.micro</span></p>
  <p>Si ves esta página, el despliegue fue un éxito.</p>
</body>
</html>
EOF
```

### 10.2. Subir al servidor con `scp`

```bash
scp -i "${KEY_NAME}.pem" \
  sitio-estatico/index.html \
  "ubuntu@${PUBLIC_IP}:/tmp/index.html"
```

### 10.3. Moverlo al directorio de Nginx

```bash
ssh -i "${KEY_NAME}.pem" "ubuntu@${PUBLIC_IP}" \
  "sudo mv /tmp/index.html /var/www/html/index.html && sudo chown www-data:www-data /var/www/html/index.html"
```

---

## 11. Verificar el sitio en el navegador

Abre tu navegador y visita:

```
http://<TU_IP_PÚBLICA>
```

O desde la terminal:

```bash
curl http://$PUBLIC_IP
```

Deberías ver el HTML desplegado. 🎉

---

## 12. Hacer lo mismo en Floci (emulador local de AWS)

> **Alternativa 100 % local y gratuita:** en lugar de lanzar una instancia real en AWS (pasos 2–11), puedes reproducir el mismo flujo con [Floci](https://floci.io), un emulador open source de AWS que corre en tu máquina. Las "instancias EC2" son **contenedores Docker reales**, así que los comandos de AWS CLI son casi idénticos: los mismos, solo que apuntando al endpoint local. No necesitas cuenta AWS, tarjeta de crédito ni credenciales reales.

### 12.1. Requisitos previos

- Docker 20.10+ y docker compose v2+ instalados
- AWS CLI v2 (el mismo que usaste en la parte de AWS)
- Una llave SSH local generada (`ssh-keygen`)

> **Diferencia clave con AWS:** Floci importa *tu llave pública real* mediante `import-key-pair`. En Floci, `create-key-pair` devolvería una llave privada ficticia que **no** sirve para SSH.

### 12.2. Instalar y arrancar Floci

**Opción A — CLI oficial:**

```bash
floci start
```

**Opción B — Docker** (montando el socket para que EC2 pueda crear contenedores):

```bash
docker pull floci/floci:latest
docker run -d --name floci \
  -p 4566:4566 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -u root \
  floci/floci:latest
```

Configura el entorno. El emulador acepta cualquier credencial no vacía:

```bash
eval $(floci env)
# equivale a:
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

Esto sustituye a los pasos 3 y 4 (configuración de credenciales de IAM y búsqueda de la AMI): aquí no hay credenciales reales ni AMIs caducables.

### 12.3. Importar la llave SSH

```bash
KEY_NAME="floci-static-site-key"

aws ec2 import-key-pair \
  --key-name "$KEY_NAME" \
  --public-key-material fileb://~/.ssh/id_rsa.pub
```

### 12.4. Red y grupo de seguridad

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
SUBNET_ID=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.1.0/24 --query 'Subnet.SubnetId' --output text)

SG_ID=$(aws ec2 create-security-group \
  --group-name "floci-static-site-sg" \
  --description "SSH (22) y HTTP (80)" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id "$SG_ID" --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0
```

### 12.5. Lanzar la instancia con UserData

Usa la AMI local `ami-ubuntu2204` (Floci la mapea a la imagen Docker `ubuntu:22.04`). A diferencia de AWS, la configuración se hace vía `--user-data` en el arranque, que en este caso instala Nginx y crea el sitio:

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ami-ubuntu2204 \
  --instance-type t2.micro \
  --key-name "$KEY_NAME" \
  --security-group-ids "$SG_ID" \
  --subnet-id "$SUBNET_ID" \
  --user-data '#!/bin/bash
apt-get update
apt-get install -y nginx
echo "¡Hola desde Floci!" > /var/www/html/index.html' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance ID: $INSTANCE_ID"
```

Equivalencias con la tabla del [paso 7](#7-lanzar-la-instancia-ec2): `ami-ubuntu2204` toma el lugar de `--image-id` (una AMI real caducable) y `--user-data` sustituye el paso manual de instalar Nginx por SSH.

### 12.6. Verificar el sitio desde localhost

No hay IP pública remota: Floci publica en tu máquina los puertos TCP abiertos en el grupo de seguridad mediante contenedores `socat`. El puerto 80 de la instancia se publica en un puerto del rango **30000–30999** (el SSH usa el rango 2200–2299). El mapeo aparece en los logs de Floci:

```
Published EC2 instance i-0abc... app port 80 on host port 30000 (socat -> 172.17.0.3:80)
```

Para descubrir el puerto publicado (o ve el sidecar `socat` con `docker ps`):

```bash
docker ps --format '{{.Names}}  {{.Ports}}'
```

Y comprueba que responde:

```bash
curl http://localhost:30000
# ¡Hola desde Floci!
```

### 12.7. Desplegar tu propio `index.html`

Del mismo modo que en el [paso 10](#10-desplegar-el-sitio-web-estático), pero copiando dentro del contenedor en lugar de `scp` al servidor remoto:

```bash
docker ps --format '{{.Names}}\t{{.Image}}'   # identifica el contenedor de la instancia
docker cp sitio-estatico/index.html <CONTAINER_NAME>:/var/www/html/index.html
docker exec <CONTAINER_NAME> nginx -s reload
```

Si prefieres el flujo SSH, conéctate al puerto SSH publicado de la instancia:

```bash
ssh -i ~/.ssh/id_rsa -p 2200 localhost
```

> Nota: la publicación de puertos SSH/HTTP y la inyección de la llave dependen de la configuración de Floci (por defecto, rango SSH 2200–2299 y puertos de app 30000–30999). Revisa los logs de Floci para confirmar el puerto real asignado.

### 12.8. Limpieza

```bash
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
floci stop      # o bien: docker stop floci
```

---

## 13. Stretch goals

### 13.1. Dominio personalizado con Amazon Route 53

1. Compra o transfiere un dominio en Route 53.
2. Crea una zona hospedada pública:

   ```bash
   aws route53 create-hosted-zone \
     --name midominio.com \
     --caller-reference "$(date +%s)"
   ```

3. Crea un registro **A** con *Alias* apuntando a la IP de la instancia (o mejor, a un ELB/CloudFront si escalas). Route 53 no soporta alias a IP cruda directamente vía CLI de forma trivial; la opción común es apuntar un registro `A` con `Value = <PUBLIC_IP>`:

   ```bash
   HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
     --dns-name "midominio.com" \
     --query 'HostedZones[0].Id' --output text | cut -d'/' -f3)

   cat > /tmp/record.json << EOF
   {
     "Changes": [{
       "Action": "CREATE",
       "ResourceRecordSet": {
         "Name": "midominio.com",
         "Type": "A",
         "TTL": 300,
         "ResourceRecords": [{"Value": "$PUBLIC_IP"}]
       }
     }]
   }
   EOF

   aws route53 change-resource-record-sets \
     --hosted-zone-id "$HOSTED_ZONE_ID" \
     --change-batch file:///tmp/record.json
   ```

4. Actualiza los *nameservers* del registrador del dominio con los que devuelve `create-hosted-zone`.

### 13.2. HTTPS con Let's Encrypt (certbot)

En el servidor:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d midominio.com -d www.midominio.com
```

Certbot:
- Solicita un certificado SSL/TLS gratuito (90 días)
- Modifica la configuración de Nginx para servir en 443
- Instala un cron de renovación automática

Verifica:

```bash
sudo certbot renew --dry-run
```

> Requiere que el dominio apunte ya a la IP de la instancia y que el puerto 80 esté abierto (para la validación HTTP-01).

### 13.3. CI/CD con AWS CodePipeline

Esquema mínimo:

1. **Repositorio de origen:** CodeCommit o GitHub (via CodeStar Connections).
2. **CodeBuild** ejecuta un build (para HTML estático, solo copiar artefactos).
3. **CodeDeploy** despliega al servidor EC2 mediante un *agent* y un `appspec.yml`:

   ```yaml
   version: 0.0
   os: linux
   files:
     - source: /
       destination: /var/www/html
   hooks:
     - location: scripts/restart-nginx.sh
       timeout: 60
       runas: root
   ```

4. En la instancia, instala el agente de CodeDeploy y crea un servicio de servicio con `trust-relationship` que permita a CodeDeploy desplegar.

Alternativa más simple para HTML puro: **CodePipeline → S3 + CloudFront** sin EC2.

---

## 14. Limpieza de recursos

Para evitar cargos fuera del Free Tier, termina todos los recursos cuando termines:

```bash
# Terminar la instancia
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"

# Eliminar el grupo de seguridad (solo después de que la instancia termine)
aws ec2 delete-security-group --group-id "$SG_ID"

# Eliminar el par de claves
aws ec2 delete-key-pair --key-name "$KEY_NAME"
rm -f "${KEY_NAME}.pem"
```

> **Nota:** El grupo de seguridad por defecto y el VPC por defecto no se eliminan; se reutilizan en futuros proyectos.

Si creaste una zona hospedada en Route 53 y ya no la necesitas:

```bash
aws route53 delete-hosted-zone --id "$HOSTED_ZONE_ID"
```

---

## 15. Resultados de aprendizaje

Al completar este proyecto habrás practicado:

- ✅ Creación y configuración de una cuenta AWS
- ✅ Uso de AWS CLI para gestionar recursos
- ✅ Diferencia entre tipos de instancias EC2 y el Free Tier
- ✅ Lanzamiento y configuración de instancias EC2
- ✅ AMIs, par de claves y grupos de seguridad
- ✅ Conexión a servidores Linux por SSH
- ✅ Administración básica de servidores (`apt`, `systemctl`)
- ✅ Instalación de Nginx y despliegue de sitios estáticos
- ✅ Exposición de servicios en la nube vía IP pública

Con estos conceptos podrás desplegar cualquiera de los proyectos anteriores (landing pages, APIs, etc.) sobre EC2, y es la base para proyectos futuros con dominios, HTTPS y CI/CD.

---

## 16. Solución de problemas

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| `Permission denied (publickey)` | Permisos de la llave incorrectos o usuario equivocado | `chmod 400 clave.pem`; usar usuario `ubuntu` |
| `UNPROTECTED PRIVATE KEY FILE` | Llave con permisos demasiado abiertos | `chmod 400 ${KEY_NAME}.pem` |
| SSH *connection timed out* | Grupo de seguridad no abre el 22 o sin IP pública | Verificar reglas ingress y `--associate-public-ip-address` |
| `curl` al puerto 80 no responde | Nginx no instalado/activo o puerto 80 cerrado | `sudo systemctl status nginx`; revisar SG |
| `aws` no encuentra credenciales | Falta `aws configure` | Ejecutar `aws configure` y verificar con `aws sts get-caller-identity` |
| `InvalidAMIID.NotFound` | AMI caducada o filtro sin resultados | Volver a ejecutar el paso 4 para obtener una AMI vigente |
| Error `InvalidGroup.NotFound` al lanzar | Grupo de seguridad creado en otro VPC | Usar el `SG_ID` devuelto en el paso 6 |

---

## Referencias

- [AWS CLI — Bash EC2 code examples](https://docs.aws.amazon.com/cli/latest/userguide/bash_ec2_code_examples.md)
- [Amazon EC2 User Guide — Connecting to your instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html)
- [Amazon EC2 — Ubuntu AMI default username](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.md)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [Nginx documentation](https://nginx.org/en/docs/)
- [Let's Encrypt / Certbot](https://certbot.eff.org/)
- [Floci — emulador local de AWS](https://floci.io/)
- [Floci — EC2 service docs](https://github.com/floci-io/floci/blob/main/docs/services/ec2.md)
- [Floci — Getting started](https://github.com/floci-io/floci/blob/main/docs/getting-started/installation.md)
