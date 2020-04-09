# Instalación de una instancia de Mastodon

Esta es una receta para instalar una instancia de Mastodon en la distribución Debian Buster del Sistema Operativo GNU/Linux.

NOTA: esta receta está hecha con fines didácticos. Una implementación seria de un servicio de red social como Mastodon requiere añadir un sistenma de tolerancia a fallos con Alta Disponibilidad y un almacenamiento seguro para los registros de la Base de Datos PostgreSQL, con replicación, respaldo y demás yerbas aromáticas.

## Requerimientos:

Esto es obligatoriamente necesario para poder comenzar a ejecutar esta receta:

- Una máquina con Debian GNU/Linux 10 (buster). Preferiblemente con 2GB de RAM, 2 procesadores y un espacio de almacenamiento adecuado a la cantidad de actividad que se espera. Se puede comenzar con unos 20GB para realizar pruebas.
- Acceso a Internet SIN PROXY para la descarga de los paquetes necesarios
- Una dirección IP pública. Aunque se puede montar en una red local privada, esta receta está orientada a la instalación de un certificado SSL (vía letsencryprt) que requiere para su verificación que el servicio pueda ser alcanzado desde internet.
- Un nombre de dominio registrado en el DNS. En este ejemplo se usará "mastodon.gnu.org.ve" y en cada línea de la receta este dominio debe sustituirse por el suyo.
- Un servicio SMTP con un usuario y contraseñas válidos. Será usado para las notificaciones vía correo electrónico.

Si alguno de los requisitos anteriores no se cumple es muy posible que no se culmine con éxito implementación del servicio.

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

Existe un script creado por los desarrolladores de Node.js para instalar el servicio en las distribuciones soportadas. Sólo ejecutarlo hará que se añadan los repositorios y sus correspondientes llaves:
```bash
curl -sL https://deb.nodesource.com/setup_10.x | bash -
```

Hecho esto la instalación del servicio se realiza de la manera acostumbrada

```bash
apt install nodejs -y
```

Luego mediante un proceso similar se instala YARN:

```bash
curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
apt update
apt install yarn -y
```

### Cambio de clave de REDIS

El servicio REDIS requiere que se haga uso de una clave de acceso. Para esto se habilita el uso de contraseñas mediante el sisguiente comando, donde será necesario sustituir "ADMINPASS" por la clave de su preferencia:

```bash
sed -i -e 's/# requirepass foobared*/requirepass ADMINPASS/' /etc/redis/redis.conf
```

Luego se accede al CLI del servicio y se realiza un primera acceso, lo cual es necesario para poder realizar peticiones posteriormente:

```bash
redis-cli
> auth ADMINPASS
> shutdown
systemctl restart redis.service
systemctl restart redis-server
```

### Creación de la Base de Datos

El servicio usa PostgreSQL como base de datos. Recuerde: esta receta es con fines didácticos, una implementación seria requiere que esta Base de Datos asegure la integridad y la réplica

```bash
su - postgres
# psql
> CREATE USER mastodon CREATEDB;
> exit
# CONTROL-D
```

## Instalación de Mastodon

Lo anterior ha servido para adaptar el sistema operativo al los requerimientos de Mastodon. Según la documentación ofocial el servicio corre deswde un usuario regular y en en ese usuario donde se instala y corre el código de la red social:

Primero, se crea el usuario, sin acceso mediante contraseña por motivos de seguridad:

```bash
adduser --disabled-login --gecos 'Mastodon Server' mastodon
```

Es importante recordar la contraseña suministrada, pues será usada mas adelante cuando toque configurar la base de datos:

Seguidamente, se accede a la cónsola de ese usuario:

```bash
su - mastodon
```

La receta oficial (y otras muchas que están en internet, sugieren la instalación de la versión 2.6.1 de RUBY, sin embargo esto actualmente genera unas inconsistencias con la última rama estable del servicio, por lo cual se usará una versión actualizada (2.6.6):

```bash
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec bash
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install 2.6.6
rbenv global 2.6.6
gem update --system --no-document
gem install bundler --no-document
```

Para probar que se ha instalado satisfactoriamente, se ejecuta el siguiente comando:
```bash
ruby --version
```

Lo cual debe arrojar una salida similar a esta:
```
ruby 2.6.6p146 (2020-03-31 revision 67876) [x86_64-linux]
```

Ahora, se descarga e instala el código de Mastodon:

```bash
git clone https://github.com/tootsuite/mastodon.git ~/live
cd ~/live
bundle install -j$(getconf _NPROCESSORS_ONLN) --deployment --without development test
```

Seguidamente se instala el manejar de paquetes YARN

```bash
yarn install --pure-lockfile
```
En algunos casos la instrucción anterior puede fallar, debido a archivos previos de YARN existentes en el repo de Mastodon. De ser así es necesario eliminarlos para proseguir la instalación y superar el error "Segmentation Fault" que se produce.

## Implementación del Servicio

Ya está todo lo necesario para implementar Mastodon. Esto se hace con la siguiente instrucción:

```bash
RAILS_ENV=production bundle exec rake mastodon:setup
```

Esto realizará un conjunto de preguntas que deben ser respondidas adecuadamente:

Nombre de Dominio
```
Domain name: mastodon.gnu.org.ve
```

Si se responde afirmativamente la siguiente pregunta, Mastodon se instalará con un único usuario y todas las peticiones se relacionarán con un único perfil. Si se desea generar una red social con mas de un usuario (lo más común) se debe responder "no" en la siguiente pregunta:
```
Do you want to enable single user mode? No
```

Si se está usando Docker, el instalador necesita saberlo. En el caso de esta receta se está usando un sistema dedicado a Mastodon, por lo tanto se responde acorde a esta condición:
```
Are you using Docker to run Mastodon? no
```

A continuación se preguntan los valores relacionados con la Base de Datos:
```
PostgreSQL host: /var/run/postgresql
PostgreSQL port: 5432
Name of PostgreSQL database: mastodon
Name of PostgreSQL user: mastodon
Password of PostgreSQL user:
```

La contraseña es la misma que se usó al crear el usuario "mastodon". Si los datos se colocaron apropiadamente, se verá el siguiente mensaje:
```
Database configuration works!
```

Seguidamente, en un proceso similar se configura la base de datos REDIS, usada para el caché de elementos en memoria:
```
Redis host: localhost
Redis port: 6379
Redis password: ADMINPASS
```


```
Redis configuration works!
```




