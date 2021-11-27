Para ejecutar contenedores debe estar instalado lo siguiente en el servidor:
- Docker
- docker-compose

Comprobar que exista una network externa, de lo contrario crear una network externa llamada web

    docker network create web


## Como montar un servicio en un subdominio con https

Verificar que el contenedor nginx y letsencrypt este activo:

    docker ps

Si no lo están, ejecutar el contenedor dentro de la carpeta nginx-proxy:

    $ cd nginx-proxy
    $ docker-compose up -d

Luego de tener los servicios activos, se puede iniciar el contenedor deseado

Para que el orquestador de dominios con HTTPS funcione es necesario agregar lo siguiente a el servicio de su archivo docker-compose

    volumes:
        # Permite al orquestador escuchar algun cambio de estado 
        - /var/run/docker.sock:/tmp/docker.sock:ro
    environment:
        # Dominio que tomará el nginx
        VIRTUAL_HOST: example.com
        # Puerto que expone la imagen
        VIRTUAL_PORT: 80
        # letsencrypt generará un certificado con este dominio
        LETSENCRYPT_HOST: example.com
        LETSENCRYPT_EMAIL: principal@shf.com.co
    networks:
          - web

Para que letsencrypt genere correctamente el certificado si no existe es necesario revisar lo siguiente:

- Verificar que el dominio en cloudflare este registrado.
- Que el dominio en cloudflare tenga el proxy desactivado (la nube debe ser gris) mientras se expide el certificado

Una vez se haya configurado el contenedor se puede iniciar

    $ docker-compose -f docker-compose.dev.yml up -d

## Errores

- **502**: Significa que existe algun error en el contenedor o no ha iniciado aun. En este caso:
> Esperar un momento o reiniciar el contenedor, puede ver los logs del contenedor usando docker logs --tail 20 -f <container-name>
- **503**: Significa que el orquestador no reconoce el dominio al cual esta ingresando como un dominio que pernezca a algun contenedor. En
 este caso:

> Verificar que los environments del contenedor esten correctos, reiniciar el contentedor


## Orquestación usando travis

En algunos casos la compilacion puede tardar un rato y necesitar mas de la cantidad de RAM existente en el servidor generando errores, en
 este caso gracias a travis podemos compilar la imagen de docker de forma remota y luego subir la imagen a algun registry de docker.

En este caso usaremos AWS Elastic Container Registry para guardar las imagenes ($0.1/GB)

Pasos para lograrlo:

### Configuración del Registry

- Crear un repo en la consola de AWS Elastic Container Registry (ECR)

- Opcional: Crear una Lifecycle policy en ECR para eliminar todas las imagenes antiguas (sin nombre)

- Verificar que este instalado aws-cli en el servidor en la version 2:

        $ aws --version
        aws-cli/2.0.19 Python/3.7.3 Linux/4.15.0-1065-aws botocore/2.0.0dev23

- Configurar las credenciales de aws en aws-cli:

    Para agregar las credenciales es necesario ir a AWS IAM, crear un nuevo usuario y vincularle la politica
     AmazonEC2ContainerRegistryFullAccess, luego agregar copias las keys en aws-cli:

        $ aws configure

- Iniciar sesión con docker en AWS en el servidor:

        aws ecr get-login-password --region <ecr_region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.us-east-1.amazonaws.com

- Usar la imagen en el docker-compose del proyecto con el nombre que aparece en el panel de AWS ECR. Ej:

        services:
          app:
            image: <account_id>.dkr.ecr.us-east-1.amazonaws.com/<repo_name>:latest


### Configuracion de Travis

- Crear el archivo .travis.yml y agregar lo siguiente:

        dist: trusty

        jobs:
          include:
            - stage: build docker image
              env:
                - AWS_ACCESS_KEY_ID=
                - AWS_SECRET_ACCESS_KEY=
                - AWS_ACCOUNT_ID=
                - IMAGE_NAME=
                - IMAGE_TAG=latest
                - AWS_REGION=us-east-1
              script:
                - docker --version  # document the version travis is using
                - pip install --user awscli # install aws cli w/o sudo
                - export PATH=$PATH:$HOME/.local/bin # put aws in the path
                - aws ecr get-login-password --region ${AWS_REGION} |
                  docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                - docker build -t ${IMAGE_NAME} .
                - docker images
                - docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}:${IMAGE_TAG}
                - docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}:${IMAGE_TAG}

- Para obtener las credenciales que se ven en el env es necesario ir a AWS IAM, crear un nuevo usuario y vincularle la politica
 AmazonEC2ContainerRegistryFullAccess


### Scripts para la automatizacion de procesos

- repull_docker.sh

        git pull
        aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 886300509739.dkr.ecr.us-east-1.amazonaws.com
        docker-compose -f docker-compose.dev.yml pull
        docker-compose -f docker-compose.dev.yml down --remove-orphans
        docker-compose -f docker-compose.dev.yml up -d
        docker image prune -f