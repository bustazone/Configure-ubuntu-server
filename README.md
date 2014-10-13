Configure-ubuntu-server
=======================

#Configurar servidor ubuntu 12.04 (apache, tomcat, mod-jk)


##Creación y configuración de Servidores Amazon
1. Una vez dentro del panel de amazon eremos a la opción de EC2 que, en efecto, son los servidores de amazon.
2. Para iniciar un servidor, en realidad hay que lanzar un instancia que se puede configurar de nuevas o lanzar una imagen creada con anterioridad.
Estas instancia mantienen los datos y cambios mientras la instancia esté creada (ya sea parada o en ejecución, no terminada) a excepción, evidentemente, de la información de la RAM.
Si queremos mantener información que permanezca aunque terminemos una instancia, tenemos dos opciones: 1- Ir creando imágenes sucesivas en la que tendremos el servidor tal cual estaba al hacer la imagen.2-Guardar la información dentro de un volumen EBS (http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumes.html) que permanecerá persistente.
3. Una vez que hemos configurado los volúmenes a utilizar (uno obligado donde correra la instancia y los demás auxiliares que, si no se dice lo contrario, no se borraran al terminar con la instancia), los grupos de seguridad y, muy importante, el par de acceso que forma una clave publica (.pem) que deberemos enviar para conectarnos mediante SSH; la instacia se lanzará. Desde ese momento podremos conectarnos a ella por medio del protocolo ssh, la dirección “public DNS” que leemos en la información de la instancia (esta dirección cambiará cada vez que paremos la instancia asi que más adelante explicaré como agregarle una IP estática desde Amazon) y el archivo .pem (aclarar que en Ubuntu el usuario al que deberemos conectarnos será “ubuntu” ya que no permite acceder directamente al root, por lo tanto, “sudo” delante de todos los mandatos  o “sudo su”).
4. Ya estamos listos para configurar el servidor.


##Configuración servidor Ubuntu 12.04

Montar volúmenes:
Es recomendable, sobre todo usando Amazon EC2 tener toda la información configurable dentro de un volumen EBS que no perdamos aunque terminemos con la instancia sin tener una imagen creada. Esto es útil por ejemplo, al crear varias intancias basadas en la misma imagen (que contendrá las aplicaciones intaladas y demás configuración común) con distintas configuraciones concretas para las aplicaciones que tengamos (Lo haremos, como veremos, enlazando mediante “soft links” los ficheros de configuración presentes en la imagen base a los archivos de configuración reales que almacenaremos en el volumen).

Este volumen EBS actúa (dentro de ubuntu) como un disco más, por lo tanto lo tendremos cargado dentro de la carpeta “dev” con el nombre que le hallamos dado al crear la instancia en la opción de enlazar volúmenes. Si por lo que sea, no encontramos esa carpeta en el directorio “dev” ejecutaremos la siguiente orden para ver que discos tenemos en el sistema :

```
[ec2-user ~]$ lsblk
NAME  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvdf  202:80   0  100G  0 disk
xvda1 202:1    0    8G  0 disk /
```

En este caso “xvda1” representa a “sda1” que es el volumen cargado en el raíz “/” y “xvdf” que representa a “sdf” que es el nombre que le hemos dado al volumen y, como vemos, de momento no está montado.
Para montarlo, en primer lugar comprobaremos que contiene un sistema de ficheros. Si contiene es que probablemente tenga información alojada y podremos montarlo directamente. Pero si no, antes de montar el volumen deberemos crearle un sistema de ficheros:

```
[ec2-user ~]$ sudo file -s /dev/xvdf
/dev/xvdf: data (No contiene un FS)
/dev/xvda1: Linux rev 1.0 ext4 filesystem data, UUID=17… (Contiene un FS)
```

En el caso de tener que crear el sistema de fichero, haremos lo siguiente (en este caso un sistema “ext4”, pero podemos usar el que sea que necesitemos):
```
[ec2-user ~]$ sudo mkfs -t ext4 device_name
```

Si ya tenemos un sistema de ficheros en el volume, crearemos un directorio que hará las veces de punto de montaje del volumen:
```
[ec2-user ~]$ sudo mkdir mount_point
```

Y por fin, podemos montar el disco:
```
[ec2-user ~]$ sudo mount device_name mount_point
```

Una vez creado, para no tener que hacer la operación de montaje cada vez que iniciemos la instancia deberemos introducir la orden de montaje en la tabla del sistemas de ficheros en “/etc/fstab”. (OJO!!! Si se introduce algo mal en fstab, el servidor puede no terminar de iniciar aunque en la instancia ponga que está iniciada, en ese caso, deberemos parar la instancia, crear otra instancia auxiliar y enlazarla a esta nueva instancia el volumen raíz de la que ha fallado para, una vez montado el volumen raíz de la instancia fallida en la nueva instancia auxiliar, podamos revertir los cambios en el archivo “fstab”).
Para añadir esta orden de montaje en “/etc/fstab” añadiremos a dicho archivo la siguiente línea separando cada dato, muy muy muy muy muy importante, por un tabulador.

**device_name  mount_point  file_system_type  fs_mntops  fs_freq  fs_passno**  

De la siguiente manera se prueba a montar todo lo que tenemos en “fstab” y así podemos comprobar errores y revertir antes de reiniciar la instancia, que no inicie el servidor y tener que hacer el truquito descrito en el “OJO!!!” del párrafo anterior
```
[ec2-user ~]$ sudo mount -a
```

En este momento, es buena idea registrar una imagen de la instancia en el panel de Amazon EC2.

Enlazar a la instancia una IP fija:
Amazon, entre sus servicios, ofrece la posibilidad de enlazar IPs fijas a las instancias lanzadas. De esta manera, aunque paremos o reiniciemos la instancia, podremos acceder (tanto via web como por ssh) a el servidor que se corre dentro de la instancia con uns simple IP fija. 
Este servicio se llama “Elastic IPs” y crear una es tan sencillo como crearla desde su pestaña en el panel de EC2 y enlazarla a la instancia que queramos.


 
Configurar servidor con postgresql, apache, tomcat, mod_jk,  ssl (en apache) :
Empezaré describiendo concretamente que necesitamos conseguir. Necesitamos desplegar una serie de aplicaciones que deben correr sobre tomcat pero que sean accesible desde apache para evitar poner el puerto (:8080) en la url. A parte necesitamos que consulten datos a una base de datos postgree. Y por último, debemos implementar seguridad SSL en apache para conseguir un acceso seguro a las aplicaciones.
	Instalar y configurar Postgresql 8.4 :
	Constatado que la versión de Postgre que necesitamos no es la la versión por defecto, debemos descargarla desde un repositorio que deberemos configurar:
Creamos el fichero “/etc/apt/sources.list.d/pgdg.list” y añadimos la siguiente linea :
	```
	 deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main 
	 ```
	 
Importamos la key del repositorio:
```
[ec2-user ~]$ wget --quiet -O -https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
```

Aparecerá un prompt (>) y escribimos:
```
> sudo apt-key add -
```

Nos dirá “OK” y actualizamos los repositorios
```
[ec2-user ~]$ sudo apt-get update
```

Antes de instalar, por si acaso, añadiremos tambien un repositorio mas pequeño propio de postgre:
```
[ec2-user ~]$ sudo add-apt-repository ppa:pitti/postgresql
```

Volvemos a actualizar los repositories :
```
[ec2-user ~]$ sudo apt-get update
```

Y, por fin, instalamos el postgresql8.4
```
[ec2-user ~]$ sudo apt-get install postgresql-8.4
```

Como hemos aclarado anteriormente, necesitaremos que los archivos susceptibles de perderse se deberán alojar en un volumen EBS. Por ello, deberemos mover el fichero donde se guardan las bases de datos al volumen y enlazarlo desde la carpeta de configuración de la siguiente manera : (NOTA!!! Este mismo proceso lo aplicaremos para la carpeta “webapps” de Tomcat, la “www” de apache o cualquier documente que no se vaya a compartir y queramos conservarlo para lincarlo más adelante )
•	En primer lugar detendremos el servidor postgre: 
```
[ec2-user ~]$ sudo service postgresql stop
```

•	Despues, crearemos una nueva carpeta en el volumen (montado en /mnt/sdf) y moveremos todo el contenido de la carpeta que actualmente aloja los archivos de la base de datos a la nueva carpeta. 
```
[ec2-user ~]$ sudo mkdir /mnt/sdf/pgBD
[ec2-user ~]$ sudo mv /var/lib/postgresql/8.2/main /mnt/sdf/pgBD
```

•	Crearemos un enlace simbólico en el directorio de base de datos antiguo para que enlace con el nuevo directorio en el volumen. 
```
[ec2-user ~]$ sudo ln –s / mnt/sdf/pgBD/main /var/lib/postgresql/8.2/main
```

•	Daremos permisos a postgre (usuario postgres) para que acceda al nuevo directorio. 
```
[ec2-user ~]$ sudo chown postgres:postgres –R /var/lib/postgresql/8.2/main
```

•	Por último, iniciamos el servicio de nuevo. 
```
[ec2-user ~]$ sudo service postgresql start
```


 
Para añadir bases de datos, usuarios , etc. usamos la aplicación gráfica “pgadmin”, para ello debemos añadir al usuario “postgres” (superusuario de postgresql) una contraseña (en este caso “postgres” tambien) y abrir las conexiones exteriores:
•	Accedemos a psql con el usuario “postgres” .
```
[ec2-user ~]$ sudo –u postgres psql
```

•	Añadimos la contraseña a dicho usuario y salimos con”\q” 
```
Postgres=# ALTER USER postgres PASSWORD 'postgres';
```

•	Para abrir la conexión, editamos el fichero de configuración “pg_hba.conf” que encontramos en “/etc/postgresql/8.4/main” añadiendo:
```
host        all         all         0.0.0.0 0.0.0.0     md5
```

•	Y editando también el fichero “postgresql.conf” editando, descomentando o añadiendo según sea necesario la línea: 
```
Listen_addresses = ‘*’
```

•	Reiniciniciamos el servicio y ya está listo para conectarnos desde el exterior con el “pgadmin”. 
```
[ec2-user ~]$ sudo service postgresql restart
```

	
 
##Instalar y configurar Apache2, Tomcat7 y mod_jk

Instalamos Apache2 :
```
[ec2-user ~]$ sudo apt-get install apache2
```

Despues, instalaremos Java6, en este, caso vamos a instalar la distribución de oracleOracle-JDK6. Para ello primero tenemos que añadir un repositorio que contiga los paquetes necesarios  y actualizar los repositorios:
```
[ec2-user ~]$ sudo add-apt-repository ppa:webupd8team/java
[ec2-user ~]$ sudo apt-get update
```

Y ya podemos instalar Java6 y podemos comprobar que la versión es correcta:
```
[ec2-user ~]$ sudo apt-get install oracle-java6-installer
[ec2-user ~]$ sudo java -version
```


Una vez tenemos java instalado, podemos instalar Tomcat7 :
```
[ec2-user ~]$ sudo apt-get install tomcat7 tomcat7-admin
```

Si nos dice que no encuentra el JDK, debemos corregir una línea del archivo “/etc/default/tomcat7”:
```
#JAVA_HOME=/usr/lib/jvm/openjdk-6-jdk
```
POR
```
JAVA_HOME=/usr/lib/jvm/java-6-oracle
```

Comprobamos que tomcat7 inicia :
```
[ec2-user ~]$ sudo service tomcat7 start
```

Por último, debemos instalar mod_jk, que conectará apache2 con tomcat :
```
[ec2-user ~]$ sudo apt-get install libapache2-mod-jk
```

En este momento ya tenemos todo instalado y podemos empezar con la configuración. 
Para ejemplificarlo de una manera clara, voy a exponer un problema:
“Tenemos dos aplicaciones web independientes que corren en tomcat (app1 y app2), además disponemos de dos dominios o subdominios (www.dominio1.com y www.dominio2.com). Necesitamos que se pueda acceder a estas aplicaciones (a app1 mediante www.dominio1.com y a app2 mediante www.dominio2.com) por la conexión del puerto 80 para no usar el puerto 8080 o donde esté ejecutando tomcat  en la url“
Para resolver este problema necestimos primeramente que las dos aplicaciones web ejecuten correctamente cada una en su dominio pero con el 8080 en la url y, a parte, que con los dos dominios salga una página servida por apache.

•	Primeramente haremos la parte de apache:
En primer lugar crearemos dentro de “/var/www” dos carpetas con nombre dom1 y dom2 que contendrán un fichero index.html muy sencillo (solo para mostrar que el servidor distigue los dominios). Estas podría ser:
```
<html>
   <head><title>dominio1</title></head>
   <body>Pagina del dominio1!!!</body>
</html>
Y
<html>
   <head><title>dominio2</title></head>
   <body>Pagina del dominio2!!!</body>
</html>
```

 
Para poder accede a cada página desde su correspondiente dominio deberemos crear un virtualhost por dominio en el fichero “/etc/apache2/sites-available/default” donde nos encontramos en un principio con: 
```
<VirtualHost *:80>
   …Configuración por defecto…
</ VirtualHost >
```

Los VirtualHost los tenemos que añadir después de la ultima etiqueta VirtualHost poniendo lo siguiente (Es un configuración básica, después se podrá añadir dentro de cada VirtualHost todo lo necesario según el caso) : 
```
<VirtualHost *:80>
DocumentRoot /var/lib/www/dom1
ServerName www.dominio1.com
 </VirtualHost>

<VirtualHost *:80>
DocumentRoot /var/lib/www/dom2
ServerName www.dominio2.com
</VirtualHost>
```

Reiniciamos apache: 
```
[ec2-user ~]$ sudo service apache2 start
```

Y con esto, ya tenemos redirigidos los dominios a nuestras páginas de apache y, por lo tanto, si ponemos www.dominio1.com iremos al index.html de dom1 y si ponemos www.dominio2.com iremos al index.html de dom2

•	Ahora necesitamos hacer algo parecido pero esta vez en Tomcat:
Igual que en el caso anterior, creamos dos carpeta en (app1 y app2) con dos sencillas aplicaciones compuestas por un “index.jsp” que puede ser como el que sigue : 
```
<html>
   <head><title>dominio<%2-1%></title></head>
   <body>Pagina del dominio<%2-1%>!!!</body>
</html>
```
Y
```
<html>
   <head><title>dominio<%1+1%></title></head>
   <body>Pagina del dominio<%1+1%>!!!</body>
</html>
```
 
Para poder acceder a cada aplicación desde <su correspondiente dominio>:8080, deberemos crear un host por dominio en el fichero “/var/lib/tomcat7/conf/server.xml” donde nos encontramos casi al final con:
```
<Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">
	…Configuracion…
</Host>
```

Los Host los tenemos que añadir después de la ultima etiqueta “Host” poniendo lo siguiente (Es un configuración básica, después se podrá añadir dentro de cada Host todo lo necesario según el caso) :
```
<Host name="www.dominio1.com" debug="0" appBase="/var/lib/tomcat7/webapps/app1" unpackWARs="true">
<Alias>www.dominio1.com</Alias>
<Logger className="org.apache.catalina.logger.FileLogger"
directory="logs" prefix="virtual_log1." suffix=".log" timestamp="true"/>
<Context path="" docBase="/var/lib/tomcat7/webapps/app1" debug="0" reloadable="true"/>
</Host>

<Host name="www.dominio2.com" debug="0" appBase="/var/lib/tomcat7/webapps/app2" unpackWARs="true">
<Alias>www.dominio2.com</Alias>
<Logger className="org.apache.catalina.logger.FileLogger"
directory="logs" prefix="virtual_log2." suffix=".log" timestamp="true"/>
<Context path="" docBase="/var/lib/tomcat7/webapps/app2" debug="0" reloadable="true"/>
</Host>
```

Reiniciamos tomcat: 
```
[ec2-user ~]$ sudo service tomcat7 start
```

Y con esto, ya tenemos redirigidos los dominios a nuestras aplicaciones de tomcat y, por lo tanto, si ponemos www.dominio1.com:8080 iremos a app1 y si ponemos www.dominio2.com iremos a app2.


•	Conexión de Apache2 a Tomcat7 a través de mod_jk:
El módulo mod_jk de apacge, utiliza el protocolo AJP para acceder a las aplicaciones de Tomcat  a través del puerto 8009. Por lo tanto, en primer lugar deberemos descomentar la siguiente línea dentro de “/var/lib/tomcat7/conf/server.xml”:
```
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```
Lo siguiente, es crearse un archivito de “workers”, éstos, serán los encargados de redirigir a los VirtualHost de apache a tomcat por el protocolo AJP. Para definir esto, crearemos un nuevo archivo llamado “workers.properties” en “/etc/apache2”. El fichero tan solo contendrá una lista de worker y unas definiciones pera cada worker. Exactamente igual que hemos dicho antes, esta configuración también es sencilla y admitirá más opciones para cada worker cuando sea necesario :
```
worker.list=dom1, dom2

worker.dom1.type=ajp13
worker.dom1.host=localhost
worker.dom1.port=8009

worker.dom2.type=ajp13
worker.dom2.host=localhost
worker.dom2.port=8009
```

Es importante asegurarse que en “/etc/apache2/mods-enabled/jk.conf” contiene la ruta correcta al “workers.properties”. Para ello hay que sustituir la línea que empieza por:
JkWorkersFile ………
Por:
```
JkWorkersFile /etc/apache2/workers.properties
```

Por último, añadiremos en los VirtualHost de apache unos puntos de montaje para indicar en qué direcciones del dominio hay que llamar al worker concreto y cambiaremos el DocumentRoot a la carpeta donde se encuentre la aplicación de tomcat:
```
<VirtualHost *:80>
DocumentRoot /var/lib/www/dom1
ServerName www.dominio1.com

JkMount /* dom1
 </VirtualHost>

<VirtualHost *:80>
DocumentRoot /var/lib/www/dom2
ServerName www.dominio2.com

JkMount /* dom2
</VirtualHost>
```
 
Seguridad:
Tenemos dos maneras de conseguir seguridad a través de SSL (Protocolo https). Podemos ponerla en Tomcat (en el puerto 8445) o podemos ponerla en apache (en el puerto 445). Evidentemente, como nuestro esquema redirige cierto tráfico de apache hacia tomcat, tiene más sentido establecer la seguridad en apache y, por supuesto, cerrar cualquier transmisión en el puerto 8080 para que no se pueda acceder a la aplicación directamente.
Para empezar vamos a revisar los elementos de seguridad necesarios. Necesitaremos:
	*Archivo del Certificado de seguridad – Archivo de extensión .crt con el certificado de seguridad del sitio creado por una agencia certificadora (si la aplicación es para uso interno valdría con un certificado creado por nosotros con openssl por ejemplo)
	*Archivo con la clave privada del certificado – Archivo que contiene la clave privada para desencriptar las transmisiones encriptadas con la clave pública. Esta clave viene a su vez cifrada (con RSA) con una contraseña dada por el que haya creado el certificado y por lo tanto al iniciar apache nos pedirá que le demos dicha contraseña. 
	*Archivo con la clave privada desencriptada – El mayor problema que tenemos con la clave privada cifrada es que al pedirnos la contraseña siempre que inicie, ante un reinicio no programado el servicio apache no iniciaría hasta que se le metieran las contraseñas de las clave privadas. Con este archivo no quitamos ese problema.

Una vez quede esto claro, vamos a instalar los módulos necesarios para que apache consiga establecer los protocolos ssl. En realidad apache ya lo trae instalado pero no activado. Por lo tanto lo activamos de la siguiente manera : 
```
[ec2-user ~]$ sudo a2enmod ssl
```

Una vez tenemos localizados los certificados y las claves podemos empezar a configurar los VirtualHost de apache para hacerlos seguros .

Para conseguir conectar a varios VirtualHost con varios certificados de seguridad tendremos que añadir la siguiente línea en “/etc/apache2/ports.conf” justo debajo de “NameVirtualHost *:80”: 
```
NameVirtualHost *:443
```

En este momento tenemos un VirtualHost para www.dominio1.com y otro para www.dominio2.com de la siguiente forma dentro del archivo “/etc/apache2/sites-available/default”:
```
<VirtualHost *:80>
DocumentRoot /var/www/dom1
ServerName www.dominio1.com

JkMount /* dom1
</VirtualHost>
```

Asi que, si queremos que ambos VirtualHost tengan seguridad ssl, comentamos estos VirtualHost y añadimos los siguientes entre dos etiquetas, que detectan si se encuentra activo el modulo ssl, justo después del último VirtualHost : 
```
<IfModule mod_ssl.c>
<VirtualHost *:443>
ServerName www.dominio1.com
ServerAlias www.dominio1.com
DocumentRoot /var/www/dom1

SSLEngine on
SSLCertificateFile    /mnt/sdf/certificados_aborda_cloud/Certdom1.crt
SSLCertificateKeyFile /mnt/sdf/certificados_aborda_cloud/KeyDecripdom1.key
BrowserMatch ".*MSIE [2-5]" nokeepalive \
ssl-unclean-shutdown downgrade-1.0 force-response-1.0

JkMount /* dom1
</VirtualHost>

<VirtualHost *:443>
ServerName www.dominio2.com
ServerAlias www.dominio2.com
DocumentRoot /var/www/dom2

SSLEngine on
SSLCertificateFile    /mnt/sdf/certificados_aborda_cloud/Certdom2.crt
SSLCertificateKeyFile /mnt/sdf/certificados_aborda_cloud/KeyDecripdom2.key
BrowserMatch ".*MSIE [2-5]" nokeepalive \
ssl-unclean-shutdown downgrade-1.0 force-response-1.0

JkMount /* dom2
</VirtualHost>
</IfModule>
```

Para clarificar donde van los archivos de certificados y claves:
```
	SSLCertificateFile    <ruta_al_archivo_del_certificado>
	SSLCertificateKeyFile     <ruta_al_archivo_de_la_clave_privade_desencriptada>
```


Nota:
La directiva  “BrowserMatch ".*MSIE [2-5]" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0” (el character \ sirva para separar una linea en varias) está adaptada de la directiva estándar “BrowserMatch ".*MSIE.*" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0”. Esta directiva solventa ciertos problemas que da Internet Explorer al conectarse a un apache con ssl. Lo he correguido por que, actualmente, estos problemas solo se darán en las versiones anteriores a InternetExplorer6 y de hecho puede interferir en el correcto funcionamiento de las versiones posteriores.
La directiva “BrowserMatch” al igual que “SetEnvIf” sirven para establecer ciertas variables de entorno cuando se el browser (en el caso de BrowserMatch) o el dato que se configure (en el caso de SetEnvIf) casen con la expresión regular que viene después.
*Directivas SetEnvIf, BrowserMatch --> http://httpd.apache.org/docs/2.2/mod/mod_setenvif.html
*Variables de entorno en apache --> http://httpd.apache.org/docs/2.2/env.html
*Apache SSL FAQ --> http://httpd.apache.org/docs/2.2/ssl/ssl_faq.html 

En este caso hemos preferido tener todos los VirtualHost en el archivo “default” pero me gustaría aclarar que cada VirtualHost puede ir en su archivo aparte dentro de “/etc/apache2/sites-available” y los seguros pueden ir en el default-ssl dentro del misma carpeta. Lo único que deberemos hacer si separamos en varios archivos es activarlos con la orden ya sean ssl o páginas normales (con esta orden lo que conseguimos es que se enlacen dentro de la carpeta “/etc/apache2/sites-enabled” y que así apache sepa que tiene que cargarlos):
```
[ec2-user ~]$ sudo a2ensite nombre_del_archivo
```

La última configuración es rechazar las peticiones que lleguen por el puerto 8080. Para ello, basta con comentar la siguiente línea en “/var/lib/tomcat7/conf/server.xml”:
```
<Connector port="8080" protocol="HTTP/1.1"
connectionTimeout="20000"
URIEncoding="UTF-8"
redirectPort="8443" />
```
 
Por ultimo, reiniciamos apache2 y tomcat7 y listo!!! 
```
[ec2-user ~]$ sudo service tomcat7 start
[ec2-user ~]$ sudo service apache2 start
```

