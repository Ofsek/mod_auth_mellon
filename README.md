#Install Apache & Dependencies

sudo apt update
sudo apt install pkg-config apache2 openssl libcurl4-openssl-dev liblasso3 liblasso3-dev build-essential apache2-dev libssl-dev

#Download & Compile & Install mod_auth_mellon from Source

wget https://github.com/latchset/mod_auth_mellon/releases/download/v0.19.1/libapache2-mod-auth-mellon_0.19.1.orig.tar.gz
sudo tar -xvzf libapache2-mod-auth-mellon_0.19.1.orig.tar.gz
cd mod_auth_mellon-0.19.1/
sudo ./configure --enable-diagnostics   
sudo make
sudo make install



#Enable Apache required modules

sudo a2enmod ssl rewrite unique_id proxy proxy_http headers unique_id substitute

sudo systemctl restart apache2



#Load mod_auth_mellon into Apache config file
sudo sed -i '1i LoadModule auth_mellon_module /usr/lib/apache2/modules/mod_auth_mellon.so' /etc/apache2/apache2.conf



#Verify Apache configs
sudo apache2ctl -M 

You should be able to see all loaded modules including mod_auth_mellon.

#Generate SP Metadata for mod_auth_mellon
#Mod_auth_mellon shipped with bash script that can create metadata automatically for you including the SP certificates
#Inside your extacted folder you will find mellon_create_metadata.sh

cd mod_auth_mellon-0.19.1/
sudo ./mellon_create_metadata.sh https://app1.ofsek.com/app1/sp https://app1.ofsek.com/app1

#The script will create three files for you, SP Metadata (https_app1.ofsek.com_app1_sp.xml), SP Certificate (https_app1.ofsek.com_app1_sp.xml), SP Private Key (https_app1.ofsek.com_app1_sp.key).
#Please make a note of the Endpoints the SLO and ACS because they going to be used with Okta.

#First let’s create a folder inside our Apache main folder to store all our SAML files.
sudo mkdir /etc/apache2/saml
#Copy and rename our SP files generated earlier from the bash script to that folder



Create Okta Apps (repeat for every app)
1. Log in to Okta Admin Console
•	Go to your Okta Admin Console (https://your-okta-domain.okta.com).
2. Create a New SAML Application
•	Navigate to Applications > Applications.
•	Click Add Application, search for SAML 2.0, and select it.
•	Click Create.
3. Configure SAML Settings
•	Single Sign-On URL: https://app1.ofsek.com/app1/postResponse   #ACS from Endpoint
•	Audience URI (SP Entity ID): https://app1.ofsek.com/app1/sp     #SP ID
4. Assign Users
•	Assign users or groups to the application by clicking Assign > Assign to People or Groups.
5. Download Okta Metadata
•	Go to Sign On, copy metadata URL and download.

#Create Apache Configuration files






For simplicity I’ve renamed the certificate and the key to be mellon.crt and mellon.key to be used with all applications.

Now we need also Okta metadata downloaded when you created the app at okta.
sudo wget -O /etc/apache2/saml/app1_idp.xml <https://okta-app1-metadata-url>

 
Now let’s see how our Apache site config will look like
sudo nano /etc/apache2/sites-available/app1.conf

<IfModule mod_ssl.c>
    <VirtualHost *:443>
        ServerName app1.ofsek.com
        SSLProxyEngine on
        ProxyPreserveHost on
        ProxyPass / http://127.0.0.1:7070/
        ProxyPassReverse / http://127.0.0.1:7070/
 
        SSLEngine on
        SSLCertificateFile /etc/ssl/certs/ssl.crt
        SSLCertificateKeyFile /etc/ssl/private/ssl.key
 
        #MellonDiagnosticsEnable On  #In case you need to troubleshoot uncomment this
        #MellonDiagnosticsFile /var/log/apache2/mellon_diagnostics.log
 
        <Location />
                MellonEnable auth
                AuthType Mellon
                MellonEndpointPath /app1/
                Require valid-user
                MellonVariable "cookie"
                MellonSecureCookie On
                MellonSessionIdleTimeout 600
                MellonSPMetadataFile /etc/apache2/saml/app1-sp.xml
                MellonSPPrivateKeyFile /etc/apache2/saml/mellon.key
                MellonSPCertFile /etc/apache2/saml/mellon.crt
                MellonIdPMetadataFile /etc/apache2/saml/app1-idp.xml
                Header always set X-Forwarded-For "%{REMOTE_ADDR}e"
                Header always set X-Forwarded-Proto "https"
                Header always set X-Forwarded-Port "443"
        </Location>
    </VirtualHost>
</IfModule>
 

The provided config file assumes that you have SSL certificates inside /etc/ssl/

