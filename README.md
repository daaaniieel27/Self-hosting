# Self-hosting

## Introducción ## 
Para realizar la práctica de self hosting primero debemos comprar un dominio de IONOS para utilizarlo al alojar la máquina host.

Procederemos a crear un archivo Vagrantfile para automatizar la creación de la máquina que vamos a utilizar:

```conf
# coding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant.configure("2") do |config|
    config.vm.box = "generic/debian11"

    config.vm.provider "virtualbox" do |vb|
      vb.gui = false
      vb.linked_clone = true
    end

    #Server
    config.vm.define "server" do |server|

      server.vm.provider "virtualbox" do |vb|
        vb.name = "server-self-hosting"
        vb.memory = "256"
      end

    server.vm.hostname= "servidor.danigg"
    server.vm.network :public_network, ip: "192.168.0.50",  hostname:  true
    server.vm.network "forwarded_port", guest: 80, host: 8080
    server.vm.network "forwarded_port", guest: 443, host: 8443
    server.vm.synced_folder  ".", "/vagrant"
    server.vm.provision "shell", inline: <<-SHELL

      # Actualizar e instalar Apache
      sudo apt update
      sudo apt-get install -y apache2
  
      # Habilitar módulos de Apache
      sudo a2enmod ssl
      sudo systemctl restart apache2
      sudo a2enmod status
      sudo systemctl restart apache2

      # Añadir certificado
      sudo cp -v /vagrant/_.danigg.com_private_key.key /etc/ssl/private
      sudo cp -v /vagrant/danigg.com_ssl_certificate.cer /etc/ssl/certs
      sudo cp -v /vagrant/ssl.conf /etc/apache2/sites-available/
      sudo a2ensite ssl.conf
      sudo systemctl restart apache2

      # Crear la estructura del sitio
      mkdir -p /var/www/html/public
      mkdir -p /var/www/html/public/Admin
      mkdir -p /var/www/html/public/Error
      sudo cp /vagrant/Servidor/Web/Inicio/index.html  /var/www/html/public/index.html
      sudo cp /vagrant/Servidor/Web/Error/404.html /var/www/html/public/Error/404.html
      sudo cp /vagrant/Servidor/Web/Admin/admin.html /var/www/html/public/Admin/admin.html
      sudo rm /var/www/html/index.html
      sudo cp -v /vagrant/danigg.com.conf /etc/apache2/sites-available
      sudo cp -v /vagrant/logo.png /var/www/html/public/

      sudo cp -v /vagrant/error404.conf /etc/apache2/sites-available/error404.conf
      sudo a2ensite error404.conf
      sudo systemctl reload apache2

      #Añadir a crontab la url para la actualización de la ip
      sudo crontab -e
      echo "5 * * * * curl https://ipv4.api.hosting.ionos.com/dns/v1/dyndns?q=MzdmMjAzMGVmZjMyNDVhNWJiMzZmZWVmMWZiNDUxNDIuVlBqVFBiUXFtMnpSbEpQLWFDOWg4aVdOWnFnOUxvblQ0LXdhb255TTdlbVlmVXpkNWVkUmdmaUJhTG8zOHBuMjAxWURXSzhkaGRaYThKT3oxMU1xWGc" >> url.txt
      sudo crontab url.txt

      #Habilitación y deshabilitación de sitios
      sudo a2dissite 000-default.conf
      sudo a2ensite danigg.com.conf
      sudo a2enmod headers
      sudo systemctl reload apache2


      # Configurar autenticación para /admin y /status
      sudo cp -v /vagrant/.htpasswd_basic /etc/apache2/.htpasswd_basic
      
      # Reiniciar Apache
      service apache2 restart
    SHELL
  end
end



```

Los detalles a destacar del archivo Vagrantfile son los siguientes:
- En la configuración de la máquina debemos añadir la línea `server.vm.network :public_network, ip: "192.168.0.50",  hostname:  true` para darle una ip que esté dentro del rango de ips del router que vayamos a utilizar para la realización de la práctica.
- Las líneas `server.vm.network "forwarded_port", guest: 80, host: 8080` y `server.vm.network "forwarded_port", guest: 443, host: 8443` están configurando un reenvío de puerto desde el puerto 80 en la máquina virtual (guest) al puerto 8080 en el sistema anfitrión (host). Esto significa que cualquier solicitud que llegue al puerto 8080 en la máquina anfitriona será redirigida al puerto 80 en la máquina virtual y la siguiente configura un reenvío de puerto similar al anterior, pero esta vez para el puerto 443 (HTTPS) en la máquina virtual al puerto 8443 en el sistema anfitrión.
- Después es importante que añadamos los comandos `sudo a2enmod ssl` y `sudo a2enmod status` para habilitar los módulos ssl para el certificado y para el módulo status, para comprobar el status de nuestro servidor. Cabe recalcar que estos módulos son módulos de apache.

## Añadir certificado ##
Para crear el certificado para conectar de manera segura he utilizado el certificado SSL de IONOS. Para implementarlo he copiado a la máquina el el archivo .key y .cer que crea para añadirlo en el siguiente archivo `ssl.conf` y que implemente el certificado.
```conf
<IfModule mod_ssl.c>
    <VirtualHost *:443>
        ServerName danigg.com
        DocumentRoot /var/www/html/public

        SSLEngine on
        SSLProtocol all -SSLv2 -SSLv3
        SSLHonorCipherOrder on
        SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
        SSLCertificateFile /etc/ssl/certs/danigg.com_ssl_certificate.cer
        SSLCertificateKeyFile /etc/ssl/private/_.danigg.com_private_key.key

    </VirtualHost>
</IfModule>
```
Es importante que habilitemos el sitio utilizando `sudo a2ensite ssl.conf`.

## Estructura del servidor web ##
Para hacer la estructura he añadido a la máquina, en la dirección root `/var/www/html/public` un archivo `index.html` y las carpetas `Error` y `Admin` con los archivos `404.html` y `admin.html` respectivamente.

### Inicio de sesión ###
Al acceder a `status` y `admin` debemos iniciar con las credenciales sysadmin:risa para status y admin:asir para admin. Tendremos que hacer un archivo de configuración y habilitar el sitio para que podamos acceder mediante las credenciales.
```conf
<VirtualHost *:80>

    ServerAdmin dani@danigg.com
    DocumentRoot /var/www/html/public
    ErrorDocument 404 /Error/404.html

    DirectoryIndex index.html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    
    <Directory "/var/www/html/public">
        Options -Indexes -Includes
        AllowOverride none
        Require all granted

    </Directory>
    
    <Directory />
        AllowOverride none
        Options -Indexes -Includes
        Require all granted          
    </Directory>
    
    <Directory "/var/www/html/public/Admin">
        AuthName "Acceso restringido"
        AuthUserFile /etc/apache2/.htpasswd_basic
        AuthType Basic
        Options Indexes FollowSymLinks
        Require user admin
    </Directory>

    <Files *.png>
        ForceType application/octet-stream
        Header set Content-Disposition "attachment"
    </Files>

    <Location /status>
        SetHandler server-status
        AuthType Basic
        AuthName "Área restringida"
        AuthUserFile /etc/apache2/.htpasswd_basic
        Require valid-user
        Require user sysadmin
    </Location>
    
</VirtualHost>
```
### Configuración error ###
En mi caso, no detecataba por defecto el archivo de error por lo que cree un archivo de configuración llamado `error404.conf` y habilitarlo para que el error que aparezca sea el personalizado:
```conf
<VirtualHost *:80>
          Alias "/public/Error" "/var/www/html/public/Error"
          <Directory "/var/www/html/public/Error">
              Options Indexes FollowSymLinks
              AllowOverride None
              Require all granted
          </Directory>
          ErrorDocument 404 /public/Error/404.html
      </VirtualHost>
```
## Actualización automática del server ##
-`sudo crontab -e`: Esto abre el editor de crontab para el usuario root. Crontab es una utilidad en sistemas Unix y Unix-like que permite programar tareas para que se ejecuten periódicamente. La opción -e significa "editar", lo que te permite editar las tareas programadas.

-`echo "5 * * * * curl https://ipv4.api.hosting.ionos.com/dns/v1/dyndns?q=MzdmMjAzMGVmZjMyNDVhNWJiMzZmZWVmMWZiNDUxNDIuVlBqVFBiUXFtMnpSbEpQLWFDOWg4aVdOWnFnOUxvblQ0LXdhb255TTdlbVlmVXpkNWVkUmdmaUJhTG8zOHBuMjAxWURXSzhkaGRaYThKT3oxMU1xWGc" >> url.txt`: Esto agrega una línea al archivo url.txt con el contenido proporcionado. El contenido es un comando curl que realiza una solicitud HTTP la URL específicada (https://ipv4.api.hosting.ionos.com/dns/v1/dyndns?q=...) para actualizar la configuración de DNS dinámico. El comando echo simplemente imprime el comando curl en la consola y >> redirige su salida (la respuesta de la solicitud HTTP) al archivo url.txt, añadiéndolo al final del archivo sin sobrescribir su contenido existente.

`sudo crontab url.txt`: Esto carga las tareas programadas desde el archivo url.txt en el crontab del usuario root. Esto significa que el comando curl que se agregó al archivo url.txt será ejecutado según la programación especificada en el crontab. La opción sudo es necesaria para realizar cambios en el crontab del usuario root.







