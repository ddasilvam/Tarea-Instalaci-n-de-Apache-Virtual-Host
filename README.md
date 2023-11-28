# Tarea: Instalación de Apache + Virtual Host
Nos basamos en el repositorio del ejercicio anterior: https://github.com/ddasilvam/Pr-ctica-1.2---DNS-Linux.
## Configuración Docker Compose
Configuramos dos contenedores: unos como servidor DNS y otro como servidor web Apache. Los colocamos ambos en la misma subred, les damos IP estáticas y hacemos que el servidor DNS se apunte a si mismo para resolver direcciones.
```
services:
  webserver_dns:
    container_name: webserver_dns
    image: ubuntu/bind9:latest
    # usamos esta imagen para poder ver los logs
    # la imagen de internetsystemsconsortium/bind9
    # no muestra los logs
    ports:
      - 53:53
    networks:
      webserver_subnet:
        ipv4_address: 172.22.0.254
    volumes:
      - ./bind9/conf:/etc/bind
      - ./bind9/zonas:/var/lib/bind
    dns:
      - 172.22.0.254
  webserver:
    image: httpd:latest
    container_name: webserver
    ports:
      - "80:80"
    volumes:
      - ./apache/htdocs:/usr/local/apache2/htdocs
      - ./apache/conf:/usr/local/apache2/conf
    networks:
      webserver_subnet:
        ipv4_address: 172.22.0.10
networks:
  webserver_subnet:
      ipam:
        driver: default
        config:
          - subnet: 172.22.0.0/24
```
## Copiamos archivos de configuración básicos a los volúmenes
Ejecutamos de forma temporal un contenedor con el servidor Apache simplemente para extraer los ficheros `httpd.conf` y `mime.types`, y los copiamos al volumen de configuración Apache en nuestro anfitrión.
```console
docker run --rm httpd:2.4 cat /usr/local/apache2/conf/httpd.conf > httpd.conf

docker run --rm httpd:2.4 cat /usr/local/apache2/conf/mime.types > mime.types
```
## Configuramos las zonas para los dominios
Creamos dos archivos de zonas para ambos dominios, `db.fabulasmaravillosas.int` y `db.fabulasoscuras.int`, y los agregamos al `named.conf.local`.
### db.fabulasmaravillosas.int
```
$TTL 38400	; 10 hours 40 minutes
@		IN SOA	ns.fabulasmaravillosas.int. some.email.address. (
				10000002   ; serial
				10800      ; refresh (3 hours)
				3600       ; retry (1 hour)
				604800     ; expire (1 week)
				38400      ; minimum (10 hours 40 minutes)
				)
@		IN NS	ns.fabulasmaravillosas.int.
ns		IN A		172.22.0.254
www	IN A		172.22.0.10
```
### db.fabulasoscuras.int
```
$TTL 38400	; 10 hours 40 minutes
@		IN SOA	ns.fabulasoscuras.int. some.email.address. (
				10000002   ; serial
				10800      ; refresh (3 hours)
				3600       ; retry (1 hour)
				604800     ; expire (1 week)
				38400      ; minimum (10 hours 40 minutes)
				)
@		IN NS	ns.fabulasoscuras.int.
ns		IN A		172.22.0.254
www	IN A		172.22.0.10
```
### Creación de un directorio para cada dominio en el volumen de htdocs
Creamos un directorio para cada unos de los dominios dentro de la carpeta del anfitrión que tenemos asignada como volumen de `htdocs` del contenedor Apache. En este caso, los llamamos `www1` y `www2`, y creamos un `index.html` en cada uno.
### Configuración de los virtual hosts en Apache
Editamos el archivo `httpd.conf` para agregar el redireccionamiento de cada dominio a cada unos de los directorios que hemos creado. Para ello, añadimos las siguientes líneas al archivo:
```
#VirtualHost 
NameVirtualHost *:80

<VirtualHost *:80>
ServerName www.fabulasmaravillosas.int
ServerAlias fabulasmaravillosas.int *.fabulasmaravillosas.int
DocumentRoot /usr/local/apache2/htdocs/www1
</VirtualHost>

<VirtualHost *:80>
ServerName www.fabulasoscuras.int
ServerAlias fabulasoscuras.int *.fabulasoscuras.int
DocumentRoot /usr/local/apache2/htdocs/www2
</VirtualHost>
```
En el DocumentRoot especificamos la ruta absoluta al directorio que contiene el `index.html` adecuado. En ServerName el nombre del dominio principal, y en ServerAlias podemos especificar subdominios del mismo.
## Comprobación del funcionamiento desde el propio servidor DNS
De esta manera podemos comprobar que el servidor DNS y el Apache funcionan correctamente. Para ello, abriendo una terminal en el servidor DNS instalamos el navegador web de consola `Lynx` usando el siguiente comando:
```console
sudo apt install lynx
```
Una vez instalado, navegamos a cada uno de los dominios y comprobamos que se muestra el `index.html` correcto. Como el servidor DNS está configurado para resolver por si mismo las direcciones, debemos poder acceder a dichos dominios. Para ello, introducimos los siguientes comandos:
```console
lynx www.fabulasmaravillosas.int

                                                                            Fabulas Maravillosas










Commands: Use arrow keys to move, '?' for help, 'q' to quit, '<-' to go back.
  Arrow keys: Up and Down to move.  Right to follow a link; Left to go back.
 H)elp O)ptions P)rint G)o M)ain screen Q)uit /=search [delete]=history list 
```
```console
lynx www.fabulasoscuras.int

                                                                               Fabulas Oscuras










Commands: Use arrow keys to move, '?' for help, 'q' to quit, '<-' to go back.
  Arrow keys: Up and Down to move.  Right to follow a link; Left to go back.
 H)elp O)ptions P)rint G)o M)ain screen Q)uit /=search [delete]=history list 
```