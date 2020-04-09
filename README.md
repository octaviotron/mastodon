# Instalación de una instancia de Mastodon

Esta es una receta para instalar una instancia de Mastodon en la distribución Debian Buster del Sistema Operativo GNU/Linux.

NOTA: esta receta está hecha con fines didácticos. Una implementación seria de un servicio de red social como Mastodon requiere añadir un sistenma de tolerancia a fallos con Alta Disponibilidad y un almacenamiento seguro para los registros de la Base de Datos PostgreSQL, con replicación, respaldo y demás yerbas aromáticas.

## Requerimientos:

Esto es obligatoriamente necesario para poder comenzar a ejecutar esta receta:

- Una máquina con Debian GNU/Linux 10 (buster). Preferiblemente con 2GB de RAM, 2 procesadores y un espacio de almacenamiento adecuado a la cantidad de actividad que se espera. Se puede comenzar con unos 20GB para realizar pruebas.
- Acceso a Internet SIN PROXY para la descarga de los paquetes necesarios
- Una dirección IP pública. Aunque se puede montar en una red local privada, esta receta está orientada a la instalación de un certificado SSL (vía letsencryprt) que requiere para su verificación que el servicio pueda ser alcanzado desde internet.
- Saber leer.


