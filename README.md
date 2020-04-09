# Instalación de una instancia de Mastodon

Esta es una receta para instalar una instancia de Mastodon en la distribución Debian Buster del Sistema Operativo GNU/Linux.

NOTA: esta receta está hecha con fines didácticos. Una implementación seria de un servicio de red social como Mastodon requiere añadir un sistenma de tolerancia a fallos con Alta Disponibilidad y un almacenamiento seguro para los registros de la Base de Datos PostgreSQL, con replicación, respaldo y demás yerbas aromáticas.

## Requerimientos:

Esto es obligatoriamente necesario para poder comenzar a ejecutar esta receta:

- Una máquina con Debian GNU/Linux 10 (buster). Preferiblemente con 2GB de RAM, 2 procesadores y un espacio de almacenamiento adecuado a la cantidad de actividad que se espera. Se puede comenzar con unos 20GB para realizar pruebas.
- Acceso a Internet SIN PROXY para la descarga de los paquetes necesarios
- Una dirección IP pública. Aunque se puede montar en una red local privada, esta receta está orientada a la instalación de un certificado SSL (vía letsencryprt) que requiere para su verificación que el servicio pueda ser alcanzado desde internet.
- Un nombre de dominio registrado en el DNS. En este ejemplo se usará "mastodon.gnu.org.ve" y en cada línea de la receta este dominio debe sustituirse por el suyo.
- Saber leer.

## Preparación del Sistema Operativo.

Después de tener una instalación mínima de Debian Buster (sin interfaz gráfica, no es para jugar buscaminas que se instalará la máquina) se asegura que se cumplen las siguientes condiciones:

###Nombre del Host FQDN

```bash
hostnamectl set-hostname mastodon.gnu.org.ve
```

Recuerda: cada vez que leas "mastodon.gnu.org.ve" debes sustituirlo por tu dominio.

###Entrada en /etc/hosts


```bash
echo "127.0.1.1 mastodon.gnu.org.ve" >> /etc/hosts
```

### Actualización del Sistema Operativo

```bash
apt update
apt upgrade -y
```

### Instalación de la paquetería base

```bash
apt install redis software-properties-common dirmngr apt-transport-https ca-certificates curl gcc g++ make imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git-core libprotobuf-dev protobuf-compiler pkg-config autoconf bison build-essential libssl-dev libyaml-dev libreadline-dev libidn11-dev libicu-dev libjemalloc-dev zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev vim certbot python-certbot-nginx nginx postgresql postgresql-contrib -y
```

## Instalación de los componentes de Mastodon

En esta sección se obtienen e instalan los componentes que requiere el servicio para funcionar:

### Node.js y YARN

```bash
curl -sL https://deb.nodesource.com/setup_10.x | bash -
apt install nodejs -y
curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
apt update
apt install yarn -y
```





