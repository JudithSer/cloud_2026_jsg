# Guía de Laboratorios Prácticos de AWS: Procedimientos Paso a Paso y Cheat Sheet

Esta guía consolida los procedimientos técnicos, comandos de terminal, scripts de automatización y directivas de configuración de los laboratorios prácticos de Amazon Web Services (AWS). Está diseñada como referencia rápida y completa para la preparación de exámenes y administración de sistemas en la nube.

---

## 1. PR0101: Preparación del Entorno y Métodos de Interacción con S3

### 1.1. Fundamentos Teóricos

La gestión en AWS se fundamenta en tres vías de interacción:

* **Consola Web de AWS (GUI)**: Entorno visual para inspección y aprendizaje inicial.
* **AWS CloudShell (CLI)**: Terminal Linux en el navegador con AWS CLI preconfigurado y autenticado.
* **SDK Python (Boto3)**: Biblioteca programática para automatizar despliegues e integraciones en scripts y contenedores.

### 1.2. Procedimiento Paso a Paso

1. **Consola Web (GUI)**:

   * Ir al servicio **S3** > **Create bucket**.
   * Asignar nombre único global (estándar DNS, ej. `asir-tunombre-web`) en región `us-east-1`.
   * Mantener bloqueo de acceso público y cifrado SSE-S3 por defecto.
   * Cargar un archivo local con el botón **Upload**.

2. **AWS CloudShell (CLI)**:

   * Comprobar credenciales de sesión:

     ```bash
     aws sts get-caller-identity
     ```

   * Crear bucket S3:

     ```bash
     aws s3 mb s3://asir-tunombre-cli --region us-east-1
     ```

   * Subir archivo local:

     ```bash
     aws s3 cp archivo.txt s3://asir-tunombre-cli/
     ```

   * Listar objetos:

     ```bash
     aws s3 ls s3://asir-tunombre-cli/
     ```

3. **Python (SDK Boto3)**:
   * Inicializar cliente y subir fichero:

     ```python
     import boto3

     # En AWS Academy se configuran las credenciales temporales o se utiliza el entorno autenticado
     s3 = boto3.client('s3', region_name='us-east-1')
     s3.create_bucket(Bucket='asir-tunombre-sdk')
     s3.upload_file('archivo.txt', 'asir-tunombre-sdk', 'archivo.txt')
     ```

---

## 2. PR0201: Despliegue de Sitio Web Estático en AWS S3

### 2.1. Fundamentos Teóricos

Amazon S3 permite alojar sitios estáticos (HTML, CSS, JS, multimedia) sin aprovisionar servidores como Apache o Nginx. Por defecto, S3 bloquea todo acceso público, por lo que se requiere habilitar el motor de hosting estático y configurar una política de bucket (Bucket Policy) pública en formato JSON.

### 2.2. Procedimiento Paso a Paso

1. **Creación del Bucket**:
   * Crear bucket con nombre único `web-asir-tunombre-ano` en región `us-east-1`.
2. **Carga de Archivos**:
   * Subir `index.html` y `error.html` a la raíz.
   * Subir la carpeta `css/` con `estilos.css`.
3. **Habilitación de Static Website Hosting**:
   * Pestaña **Properties** > **Static website hosting** > **Edit** > **Enable**.
   * *Hosting type*: Host a static website.
   * *Index document*: `index.html`.
   * *Error document*: `error.html`.
   * Guardar cambios y copiar el **Endpoint URL** del sitio web generado.
4. **Desbloqueo de Acceso Público**:
   * Pestaña **Permissions** > **Block public access (bucket settings)** > **Edit**.
   * Desmarcar la casilla principal **Block all public access** > **Save changes** > Confirmar escribiendo `confirm` o `confirmar`.
5. **Aplicación de la Bucket Policy**:
   * En la sección **Bucket policy** > **Edit** > Pegar la siguiente política JSON:

*Restricción opcional por dirección IP pública*:
Para restringir el acceso a una IP específica, se añade la condición dentro del Statement:

```json
"Condition": {
  "IpAddress": {
    "aws:SourceIp": "TU_IP_PUBLICA/32"
  }
}
```

---

## 3. PR0202: Automatización de Operaciones en S3 con Python (Boto3)

### 3.1. Fundamentos Teóricos

El SDK Boto3 permite interactuar mediante API con todos los servicios de AWS. La gestión de buckets y objetos requiere un manejo riguroso de excepciones (`botocore.exceptions.ClientError`) para capturar colisiones de nombres (`BucketAlreadyExists`) y errores de acceso. Además, un bucket no puede eliminarse si contiene objetos; debe vaciarse previamente.

### 3.2. Operaciones y Scripts Esenciales

* **Verificación de Identidad con STS**:

  ```python
  import boto3
  sts = boto3.client('sts')
  identidad = sts.get_caller_identity()
  print(f"Cuenta: {identidad['Account']} | ARN: {identidad['Arn']}")
  ```

* **Creación de Bucket con Captura de Colisiones**:

  ```python
  from botocore.exceptions import ClientError
  import boto3

  s3 = boto3.client('s3', region_name='us-east-1')
  bucket_name = 'mi-bucket-automatizado-2026'

  try:
      s3.create_bucket(Bucket=bucket_name)
      print(f"Bucket '{bucket_name}' creado con exito.")
  except ClientError as e:
      codigo = e.response['Error']['Code']
      if codigo in ('BucketAlreadyExists', 'BucketAlreadyOwnedByYou'):
          print(f"Aviso: El nombre '{bucket_name}' ya existe en AWS.")
      else:
          raise e
  ```

* **Ingesta con Metadatos y Tipo MIME**:

  ```python
  s3.upload_file(
      Filename='documento.pdf',
      Bucket=bucket_name,
      Key='informes/2026/documento.pdf',
      ExtraArgs={
          'ContentType': 'application/pdf',
          'Metadata': {'departamento': 'sistemas', 'entorno': 'lab'}
      }
  )
  ```

* **Auditoría y Listado con Prefijos**:

  ```python
  respuesta = s3.list_objects_v2(Bucket=bucket_name, Prefix='informes/')
  if 'Contents' in respuesta:
      for obj in respuesta['Contents']:
          print(f"Clave: {obj['Key']} | Tamano: {obj['Size']} bytes | Modificado: {obj['LastModified']}")
  else:
      print("No se encontraron objetos con ese prefijo.")
  ```

* **Descarga de Objetos**:

  ```python
  s3.download_file(Bucket=bucket_name, Key='informes/2026/documento.pdf', Filename='documento_local.pdf')
  ```

* **Vaciado y Eliminación del Bucket**:

  ```python
  # 1. Eliminar todos los objetos presentes
  objetos = s3.list_objects_v2(Bucket=bucket_name)
  if 'Contents' in objetos:
      for obj in objetos['Contents']:
          s3.delete_object(Bucket=bucket_name, Key=obj['Key'])

  # 2. Eliminar el contenedor vacio
  s3.delete_bucket(Bucket=bucket_name)
  print(f"Bucket {bucket_name} eliminado completamente.")
  ```

---

## 4. PR0203: Resiliencia con Versionado y Ciclo de Vida en Amazon S3

### 4.1. Fundamentos Teóricos

* **Versionado**: Conserva todas las versiones de cada objeto almacenado. Al sobreescribir un archivo, se crea una nueva versión con `VersionId` único.
* **Marcadores de Borrado (Delete Markers)**: Al ejecutar un borrado simple (`aws s3 rm`), el archivo no se destruye; S3 inserta un marcador de borrado como versión activa actual. El archivo se restaura eliminando dicho marcador.
* **Políticas de Ciclo de Vida (Lifecycle Rules)**: Automatizan la transición de objetos y versiones no actuales a clases de almacenamiento económico (`STANDARD_IA`, `GLACIER`) y su posterior expiración.

### 4.2. Procedimiento y Comandos CLI

* **Habilitar el Versionado**:

  ```bash
  aws s3api put-bucket-versioning --bucket NOMBRE-BUCKET --versioning-configuration Status=Enabled
  ```

* **Recuperar una Versión Sobreescrita**:
  1. Listar el historial de versiones:

     ```bash
     aws s3api list-object-versions --bucket NOMBRE-BUCKET --prefix ruta/archivo.txt
     ```

  2. Descargar la versión previa específica:

     ```bash
     aws s3api get-object --bucket NOMBRE-BUCKET --key ruta/archivo.txt --version-id "ID_VERSION" archivo_recuperado.txt
     ```

* **Restaurar un Objeto Borrado (Eliminar Delete Marker)**:
  1. Localizar el marcador de borrado:

     ```bash
     aws s3api list-object-versions --bucket NOMBRE-BUCKET --query "DeleteMarkers[?Key=='ruta/archivo.txt']"
     ```

  2. Eliminar el marcador de borrado para que la versión previa vuelva a ser la activa:

     ```bash
     aws s3api delete-object --bucket NOMBRE-BUCKET --key ruta/archivo.txt --version-id "ID_DELETE_MARKER"
     ```

* **Configuración de Políticas de Ciclo de Vida**:

Aplicar la regla al bucket:

```bash
aws s3api put-bucket-lifecycle-configuration --bucket NOMBRE-BUCKET --lifecycle-configuration file://lifecycle_rules.json
```

---

## 5. PR0301: EC2, Automatización con User Data, IMDSv2 y Security Groups

### 5.1. Fundamentos Teóricos

* **User Data (cloud-init)**: Script de inicialización que se ejecuta con permisos de superusuario (`root`) **una sola vez durante el primer arranque** de la instancia.
* **IMDSv2 (Instance Metadata Service v2)**: API interna de enlace local (IP `169.254.169.254`). Requiere solicitar obligatoriamente un token de sesión temporal mediante método `PUT` con cabecera `X-aws-ec2-metadata-token-ttl-seconds` antes de consultar los metadatos con `GET`.
* **Security Groups**: Cortafuegos virtual **con estado (stateful)** a nivel de interfaz de red (ENI). Si se autoriza el tráfico entrante en un puerto, la respuesta saliente se permite de forma automática.

### 5.2. Procedimiento de Configuración

1. **Security Group**:
   * *Inbound*: HTTP (80) abierto a `0.0.0.0/0` y SSH (22) restringido a la IP del administrador (`TU_IP/32`).
   * *Outbound*: Todo el tráfico permitido (`0.0.0.0/0`).
2. **Script de User Data**:

    ```bash
    #!/bin/bash
    # -----------------------------------------------------------------
    # 1. Actualización e instalación del servidor web Apache
    # -----------------------------------------------------------------
    dnf update -y
    dnf install -y httpd

    # Habilitar e iniciar el servicio en el arranque de systemd
    systemctl enable httpd
    systemctl start httpd

    # -----------------------------------------------------------------
    # 2. Obtención de metadatos mediante IMDSv2
    # -----------------------------------------------------------------
    # Paso A: Generar token de sesión válido por 6 horas (21600 segundos)
    TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
      -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

    # Paso B: Consultar metadatos usando el token en la cabecera HTTP
    INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/instance-id)

    INSTANCE_TYPE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/instance-type)

    AVAILABILITY_ZONE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/placement/availability-zone)

    PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/local-ipv4)

    # -----------------------------------------------------------------
    # 3. Creación del archivo index.html dinámico
    # -----------------------------------------------------------------
    cat <<EOF > /var/www/html/index.html
    <!DOCTYPE html>
    <html lang="es">
    <head>
        <meta charset="UTF-8">
        <title>Servidor Web EC2 - ASIR Cloud</title>
        <style>
            body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #f4f6f9; margin: 40px; color: #333; }
            .card { background: white; max-width: 650px; margin: 0 auto; padding: 25px; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); border-top: 5px solid #FF9900; }
            h1 { color: #232F3E; font-size: 24px; margin-bottom: 20px; }
            table { width: 100%; border-collapse: collapse; margin-top: 15px; }
            th, td { text-align: left; padding: 12px; border-bottom: 1px solid #e1e4e8; }
            th { background-color: #f8f9fa; color: #555; }
            .badge { background: #e8f5e9; color: #2e7d32; padding: 4px 8px; border-radius: 4px; font-weight: bold; font-family: monospace; }
        </style>
    </head>
    <body>
        <div class="card">
            <h1>Servidor Web Aprovisionado con User Data</h1>
            <p>Estado del servicio: <span class="badge">ACTIVO (HTTPD)</span></p>
            <table>
                <tr><th>Parámetro</th><th>Valor consultado (IMDSv2)</th></tr>
                <tr><td><strong>ID de la Instancia</strong></td><td>$INSTANCE_ID</td></tr>
                <tr><td><strong>Tipo de Instancia</strong></td><td>$INSTANCE_TYPE</td></tr>
                <tr><td><strong>Zona de Disponibilidad (AZ)</strong></td><td>$AVAILABILITY_ZONE</td></tr>
                <tr><td><strong>Dirección IP Privada</strong></td><td>$PRIVATE_IP</td></tr>
            </table>
        </div>
    </body>
    </html>
    EOF


    ```

3. **Lanzamiento de la Instancia**:
   * SO: Amazon Linux 2023, tipo `t2.micro` o `t3.micro`.
   * En *Advanced details*: fijar **Metadata version** en **V2 only (token required)** y pegar el script en **User data**.
4. **Diagnóstico y Troubleshooting**:
   * Si la página no carga, conectarse por SSH y comprobar el log de cloud-init:

     ```bash
     cat /var/log/cloud-init-output.log
     ```

---

## 6. PR0302: Gestión de Volúmenes EBS Secundarios, `/etc/fstab` y AMIs

### 6.1. Fundamentos Teóricos

* **Amazon EBS**: Almacenamiento en bloque en red con persistencia independiente del ciclo de vida de la instancia. **Un volumen EBS está estrictamente confinado a su Zona de Disponibilidad (AZ)** y solo puede conectarse a instancias en esa misma AZ.
* **Volúmenes gp3**: Permiten provisionar IOPS y rendimiento (throughput) de manera independiente al tamaño del disco.
* **Hipervisor Nitro y Dispositivos NVMe**: En instancias Nitro, Linux mapea los volúmenes de bloques como dispositivos NVMe (`/dev/nvme1n1`) en lugar de los nombres tradicionales SCSI (`/dev/xvdf` o `/dev/sdf`).
* **Peligro de Boot Loop en `/etc/fstab`**: Si un disco montado en `/etc/fstab` falla o no está presente al reiniciar, Linux se detiene en modo de emergencia. Para evitarlo, es obligatorio usar el **UUID** y la opción **`nofail`**.

### 6.2. Procedimiento de Creación, Formateo y Montaje

1. **Crear y Conectar Volumen**:
   * En consola EC2 > **Elastic Block Store > Volumes > Create volume** (tipo `gp3`, misma AZ que la instancia).
   * Seleccionar volumen > **Actions > Attach volume** > asociar a la instancia con dispositivo `/dev/sdf`.
2. **Comandos en Linux (SSH)**:
   * Identificar discos:

     ```bash
     lsblk
     ```

   * Comprobar formato del disco:

     ```bash
     sudo file -s /dev/nvme1n1
     # Si devuelve "data", el disco está virgen y requiere formato.
     ```

   * Formatear en ext4:

     ```bash
     sudo mkfs -t ext4 /dev/nvme1n1
     ```

   * Crear directorio y montar temporalmente:

     ```bash
     sudo mkdir -p /mnt/datos
     sudo mount /dev/nvme1n1 /mnt/datos
     ```

   * Obtener el identificador único UUID:

     ```bash
     sudo blkid /dev/nvme1n1
     ```

3. **Configuración de Persistencia en `/etc/fstab`**:
   * Editar `/etc/fstab`:

     ```bash
     sudo nano /etc/fstab
     ```

   * Añadir al final del archivo:

     ```text
     UUID=tu-uuid-obtenido    /mnt/datos    ext4    defaults,nofail    0    2
     ```

   * **Validación obligatoria sin reiniciar**:

     ```bash
     sudo umount /mnt/datos
     sudo mount -a
     df -h /mnt/datos
     ```

4. **Clonación mediante AMIs**:
   * Seleccionar la instancia configurada > **Actions > Image and templates > Create image**.
   * Una vez en estado *Available*, lanzar una nueva instancia a partir de la AMI: el servidor clonado contendrá idéntico software, datos y configuraciones.

---

## 7. PR0303: Arquitectura Redundante con ALB y Auto Scaling

### 7.1. Fundamentos Teóricos

* **Application Load Balancer (ALB)**: Balanceador en Capa 7 (HTTP/HTTPS) que distribuye peticiones entrantes entre instancias situadas en múltiples Zonas de Disponibilidad.
* **Encadenamiento de Security Groups (Security Group Chaining)**: Técnica de endurecimiento de seguridad donde las instancias backend solo autorizan tráfico procedente del Security Group del balanceador, bloqueando cualquier intento de conexión directa desde Internet.
* **Auto Scaling Group (ASG)**: Mantiene la alta disponibilidad y la elasticidad horizontal aprovisionando o destruyendo instancias dinámicamente según la carga de CPU.

### 7.2. Procedimiento Paso a Paso

1. **Configuración de Security Groups Encadenados**:
   * `sg-alb-publico`: Inbound HTTP (80) abierto a `0.0.0.0/0`.
   * `sg-ec2-backend`: Inbound HTTP (80) con **Source = `sg-alb-publico`** (bloquea el acceso público directo). Añadir regla SSH (22) para administración.
2. **Plantilla de Lanzamiento (Launch Template)**:
   * Crear plantilla con Amazon Linux 2023, `t2.micro` o `t3.micro`, Security Group `sg-ec2-backend`, IMDSv2 requerido y script User Data que instale `httpd` y la herramienta `stress`.
3. **Grupo de Destino (Target Group)**:
   * Target type: **Instances**, puerto HTTP 80, Health Checks en `/`. **No registrar instancias manualmente.**
4. **Application Load Balancer (ALB)**:
   * Esquema: **Internet-facing**, seleccionar **al menos 2 Zonas de Disponibilidad (AZs)** públicas, asociar `sg-alb-publico` y configurar el Listener HTTP 80 para reenviar (*Forward*) al Target Group.
5. **Auto Scaling Group (ASG)**:
   * Seleccionar la Launch Template y las subredes del balanceador.
   * En *Advanced options*: Asociar al Target Group existente y marcar **Turn on Elastic Load Balancing health checks** (Periodo de gracia: 120-300 s).
   * Capacidades: Deseada = 2, Mínima = 2, Máxima = 4.
   * Política de escalado: *Target tracking scaling policy* basada en utilización media de CPU al 50%.
6. **Pruebas y Verificación**:
   * Comprobar distribución Round-Robin:

     ```bash
     for i in {1..10}; do curl -s http://DNS-DEL-BALANCEADOR/ | grep "ID Instancia"; done
     ```

   * Simular sobrecarga de CPU por SSH para forzar el autoescalado:

     ```bash
     sudo stress --cpu 2 --timeout 480s
     ```

---

## 8. PR0304: Contenedores Serverless con Amazon ECS sobre AWS Fargate

### 8.1. Fundamentos Teóricos

* **Container as a Service (CaaS)**: Despliegue de aplicaciones empaquetadas en contenedores Docker sin administrar la infraestructura subyacente.
* **AWS Fargate**: Motor de cómputo serverless para Amazon ECS. Abstrae por completo las máquinas virtuales EC2: el usuario solo define vCPU y memoria requeridas por tarea.
* **Arquitectura de ECS**:
  * *Clúster*: Límite lógico de la infraestructura.
  * *Task Definition*: Plano descriptivo inmutable (imagen Docker, puertos, variables, rol IAM).
  * *Task*: Instancia única de un contenedor en ejecución.
  * *Service*: Mantiene el número deseado de réplicas en ejecución y gestiona la autorrecuperación (*self-healing*).

### 8.2. Procedimiento Paso a Paso

1. **Creación del Clúster ECS**:
   * En Amazon ECS > **Clusters > Create cluster** > Nombre `cluster-asir-fargate` > Seleccionar exclusivamente **AWS Fargate (serverless)**.
2. **Definición de Tarea (Task Definition)**:
   * En **Task Definitions > Create new task definition**:
     * Launch type: **AWS Fargate**, Linux/X86_64.
     * Recursos: CPU `0.25 vCPU`, Memoria `0.5 GB`.
     * **Task execution role**: Seleccionar obligatoriamente **`LabRole`**.
     * Contenedor: Nombre `web-nginx`, imagen `nginx:alpine`, mapeo de puerto `80/tcp`.
3. **Despliegue del Servicio**:
   * En la Task Definition creada: **Deploy > Create service**.
   * Clúster: `cluster-asir-fargate`, Service type: Replica, **Desired tasks: 2**.
   * Red: Subredes públicas en múltiples AZs, Security Group con puerto 80 abierto a `0.0.0.0/0` y **Auto-assign public IP: Turn on**.
4. **Verificación y Autorrecuperación (Self-Healing)**:
   * En la pestaña **Tasks**, copiar la IP pública de una tarea y verificar la web de Nginx en el navegador.
   * **Prueba de fallo**: Seleccionar una tarea y pulsar **Stop task**. El servicio detectará la discrepancia con la capacidad deseada y desplegará de inmediato una nueva tarea para restituir las 2 réplicas.

---

## 9. PR0305: Cómputo Orientado a Eventos con AWS Lambda y S3

### 9.1. Fundamentos Teóricos

* **Serverless (FaaS)**: Paradigma de computación donde el código se ejecuta únicamente en respuesta a eventos del sistema. Escala a cero y no genera costes mientras no existan peticiones activas.
* **Arquitectura Orientada a Eventos**: La subida de un documento a un bucket S3 genera una notificación asíncrona estructurada en formato JSON (`s3:ObjectCreated:*`) que activa automáticamente la ejecución de la función Lambda.
* **Seguridad con LabRole**: La función Lambda asume un rol de ejecución IAM (`LabRole`) para obtener privilegios temporales de lectura sobre S3 y escritura de trazas en Amazon CloudWatch Logs.

### 9.2. Procedimiento Paso a Paso

1. **Creación del Bucket S3 de Ingesta**:
   * Crear bucket privado en `us-east-1` (ejemplo: `asir-eventos-tunombre`).
2. **Creación de la Función Lambda**:
   * En AWS Lambda > **Create function** > **Author from scratch**.
   * Nombre: `procesador-archivos-s3`, Runtime: Python 3.11, Arquitectura: x86_64.
   * Permisos: Marcar **Use an existing role** y elegir estrictamente **`LabRole`**.
3. **Código de la Función (Python con Boto3)**:

4. **Configuración del Disparador (Trigger S3)**:
   * En la función Lambda > **Add trigger** > Origen: **S3**.
   * Seleccionar el bucket creado > Event type: **All object create events (`s3:ObjectCreated:*`)**.
   * Aceptar la advertencia de recursividad y pulsar **Add**.
5. **Validación y Monitorización en CloudWatch Logs**:
   * Cargar un archivo `.txt` en el bucket S3.
   * En la función Lambda > Pestaña **Monitor** > **View CloudWatch logs**.
   * Acceder al último *Log Stream* para verificar la traza de ejecución con el nombre del archivo y el cómputo de líneas procesadas.
