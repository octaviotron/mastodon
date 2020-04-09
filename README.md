# Instalación de una instancia de Mastodon

Licencia CC-BY-SA o FDL

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

### Configuración de REDIS

Mastodon accede a REDIS autenticando mediante una clave. Siguiendo las instrucciones de la documentación oficial (y de otras consultadas en internet) el instalador arroja un mensaje de error, puesto que en ningún paso anterior se define u obtiene dicha contraseña y no es posible ingresar los valores apropiados que solicita el script de instalación.

Este conjunto de comandos logró superar con éxito la conexión con este servicio mas adelante donde se verifica y configura este servicio:

```bash
sed -i -e 's/# requirepass foobared*/requirepass ADMINPASS/' /etc/redis/redis.conf
```
Se accede entonces al CLI del servicio:
```bash
redis-cli
```

Se verá la línea de comandos precedida con el signo ">" con lo cuasl se sabe que se está dentro del CLI de Redis. Allí se ejecutan estos dos comandos:

```
> auth ADMINPASS
> shutdown
```

Con lo anterior se genera un primer acceso lo cual según la documentación permite posteriormente acceder usando esa contraseña como credencial.

NOTA: es posible que se pueda omitir la modificación del archivo de configuración "/etc/redis/redis.conf" que se realizó en el paso anterior y sólo ejecutar las instrucciones dentro del CLI de Redis. Esto queda por verificar y de ser cierto mejora la seguridad al no quedar en un archivo de texto plano la contraseña del servicio.

Para que Redis aplique los cambios, se reinicia el servicio:
```bash
systemctl restart redis.service
systemctl restart redis-server
```

( NOTA: es posible que se pueda omitir la modificación del archivo de configuración "/etc/redis/redis.conf" y sólo ejecutar las instrucciones dentro del CLI de Redis. Por verificar )

### Configuración de la Base de Datos

El servicio usa PostgreSQL como base de datos. Recuerde: esta receta es con fines didácticos, una implementación seria requiere que este servicio pueda garantizar la integridad y el respaldo de datos en un ambiente de producción con usuarios reales.

```bash
su - postgres
```

Se accede a la CLI de PostgreSQL:
```
psql
```

y dentro se crea la base de datos "mastodon":
```
CREATE USER mastodon CREATEDB;
exit
```

Para salir del usuario postgres se presiona la combinación CONTROL-D o se ejecuta nuevamente exit

```bash
exit
```

Lo importante es asegurarse que en este punto se está de nuevo como root en la cónsola.

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

Mastodon usa centralmente el lenguaje RUBY, el cual requiere tener un entorno que se genera con los siguiente pasos: 

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

Los primeros valores se seleccionan presionando ENTER, pues se sugieren por defecto. En esta receta la base de datos se llama "mastodon" (distinto del valor por defecto). El usuario debe ser "mastodon" y la contraseña es la misma que se usó al crear ese usuario. 

Si los datos se colocaron apropiadamente, se verá el siguiente mensaje:
```
Database configuration works!
```

Es muy necesario recordar que en un ambiente de producción con usuarios reales se debe contemplar la integridad y respaldo de la base de datos usando las técnicas conocidas para tal fin.

Si se comete un error al suministrar los datos, el instalador permitirá reitentar o continuar. No se debe continuar sin lograr una conexión satisfactoria.

Seguidamente, en un proceso similar se configura la base de datos REDIS, usada para el caché de elementos en memoria:
```
Redis host: localhost
Redis port: 6379
Redis password: ADMINPASS
```

Si se suministran los datos correctamente (los dos primeros se seleccionan presionando ENTER para aceptar los que se sugieren por defecto)
```
Redis configuration works!
```

Al igual que la configuración anterior, es posible repetir este paso hasta lograr la conexión, sin lo cual no se debe continuar o todo el proceso fallará posteriomente.

Luego el instalador preguntará si los archivos que suben los usuarios serán almacenados en una nube. En esta receta se responde "no", pero es posible posteriormente configurarlo desde la interfaz gráfica para almacenar los archivos multimedia en un servicio externo (recomendable en un ambiente de producción):

```
Do you want to store uploaded files on the cloud? No
```

A continuación se pedirán los datos para el envío de mensajes vía correo electrónico. Esto es muy necesario y se requiere conocer el sistema de autenticación, tener acceso y lograr autorización en su servidor SMTP. A continuación se muestra un ejemplo que debe ser ajustado insertando los valores apropiados que como es lógico deberá cambiar para que se correspondan con los de su servicio:

```
Do you want to send e-mails from localhost? No
SMTP server: smtp.gnu.org.ve
SMTP port: 587
SMTP username: mastodon-admin@gnu.org.ve
SMTP password:
SMTP authentication: plain
SMTP OpenSSL verify mode: none
E-mail address to send e-mails "from": mastodon-admin@gnu.org.ve
Send a test e-mail with this configuration right now? Yes
Send test e-mail to: octavio.rossell@gmail.com
```

De forma similar, el instalador permitirá hacer varios intentos en caso de que no logre enviar satisfactoriamente un correo. Es muy importante que se logre exitosamente y verificar que se ha recibido, preferiblemente usando un destinatario externo al dominio (en este ejemplo se hace la prueba con un buzón en gmail)

Es posible guardar la configuración en un archivo para tener el registro de los valores suministrados que se guarda como "~/.env.production", para eso se responde afirmativamente en la próxima pregunta:
```
Save configuration? Yes
```

El esquema de la base de datos está listo en este punto para ser desplegado en el servicio de PostgreSQL:
```
Prepare the database now? Yes
```

Posteriormente el instalador pregunta si se desea compilar los componentes de Javascript y CSS, advirtiendo que esto requerirá usar memoria ram y que requerirá algo de tiempo. Es preferible hacerlo, ya que de lo contrario quedará incompleto el sistema:
```
Compile the assets now? (Y/n) Yes
```

Finalmente, se debe crear el usuario administrador, el cual tendrá acceso en la interfaz gráfica a la configuración del servidor de Mastodon, al sistema de reportes y a otros módulos adicionales. Por supuesto deben ajustarse los valores que se muestran como ejemplo:
```
Do you want to create an admin user straight away? Yes
Username: admin
E-mail: mastodon-admin@gnu.org.ve
```

El proceso terminará mostrando un mensaje con una contraseña para acceder como administrador de la instancia de Mastodon, que será necesario cambiar al entrar por primera vez en la sesión de usuario:
```
You can login with the password: 71bd1d728ddc1950a352deadae0a7a25
```

## Habilitación del Servicio

Ya Masstodon está instalado. Ahora hay que crear, levantar y dejar habilitado el servicio.

Se copian los archivos con las entradas necesarias para Systemd que están disponibles dentro de lo que se descargó de Mastodon via GIT:
```
cp /home/mastodon/live/dist/mastodon-web.service /etc/systemd/system/
cp /home/mastodon/live/dist/mastodon-sidekiq.service /etc/systemd/system/
cp /home/mastodon/live/dist/mastodon-streaming.service /etc/systemd/system/
```

EL último paso consiste en levantar los servicios y dejarlos habilitados para que se carguen cada vez que se inicie el sistema operativo:
```
systemctl start mastodon-web
systemctl start mastodon-sidekiq
systemctl start mastodon-streaming
systemctl enable mastodon-web
systemctl enable mastodon-sidekiq
systemctl enable mastodon-streaming
```

Para acceder a la instancia instalada de esta red social, se abre un navegador en una URL con el dominio que se usó en su intalación:

```
http://mastodon.gnu.org.ve
```

## Conclusión

Mastodon emplea diversas piezas de software para funcionar. En primer lugar el lenguaje de programación RUBY se encarga de la parte central de la lógica del servicio. La otra parte importante se compone de código en Javascript, el cual se provee mediante Node.js. PostgreSQL se usa para acceder a la base de datos general del sistema, junto a Redis quien provee un caché para optimizar la lectura de información por parte de algunos componentes del servicio.

Construyendo esta receta pude notar que al igual que en muchos otros desarrollos en los cuales se emplea el lenguaje de programación RUBY, hay una estricta condición de funcionamiento relacionada a las versiones de los componentes del lenguaje y sus módulos (llamados gemas) y de ellos respecto a su interoperabilidad con el resto de los programas con los que se relaciona dentro del sistema y añadadido a eso la instalación está condicionada para sólo funcionar en cierta arquitectura de hardware: si se observa la sección con las instrucciones para instalar RUBY, se podrá ver que se realiza la compilación de un binario que por tanto estará estáticamente ligado al la arquitectura de procesador donde se ejecuta.

```bash
cd ~/.rbenv && src/configure && make -C src
```

La intención de tener esta receta partió de la sugerencia de un colega para crear una ISO instalable que pudiera implementar de forma sencilla una instancia de Mastodon, pero esa motivación resultó en esta guía que con solo "copiar y pegar" (literalmente) brinda un método para tener el servicio montado en una máquina con un Debian Buster fresco.

Como posdata, la reiteración acerca de que esta receta se debe complementar con pasos para incluir la integridad y el respaldo de la base de datos, indispensable para plantearse un ambiente de producción con usuarios reales, así como la opcional pero necesaria aplicación de sistemas que provean Alta Disponibilidad del servicio y Balanceo de Carga o proxeo de peticiones desde los clientes.

Happy Hacking
