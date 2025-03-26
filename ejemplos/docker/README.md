
# Instalar una BBDD: MariaDB

Qué parametros/configuración mínima tengo que hacer?
- Usuario                           usuario
- Contraseña                        password
- Contraseña root                   password
- Nombre instancia (BBDD)           bd
- Puerto: 3306

La BBDD guardará datos. Querremos que esos datos se pierdan tras el borrado del contenedor? NO
Nos hará falta guardar los datos fuera del contenedor (VOLUMEN). Lo vamos a crear a nivel del host:
    /home/ubuntu/environment/datos/bd
    
De paso, vamos a ver si montamos un cliente web de BBDD: ADMINER
El Adminer como cliente WEB, abre un puerto HTTP. Con independencia de cuál abra internamente (a nivel del contenedor), 
vamos a exponer (NAT) ese servicio en el puerto 8080 del host