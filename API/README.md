# How to Generate Certs In Ubuntu

## On Ubuntu the standard mechanism would be:

* dotnet dev-certs https -v to generate a self-signed cert


        cd path/to/project
        dotnet dev-certs https -v


* convert the generated cert in ~/.dotnet/corefx/cryptography/x509stores/my from pfx to pem using


        openssl pkcs12 -in ~/.dotnet/corefx/cryptography/x509stores/my/<certname>.pfx -nokeys -out localhost.crt -nodes


* copy localhost.crt to /usr/local/share/ca-certificates


        sudo cp -v localhost.crt /usr/local/share/ca-certificates/localhost.crt


* trust the certificate using


        sudo update-ca-certificates


* verify if the cert is copied to /etc/ssl/certs/localhost.pem (extension changes)


        ls /etc/ssl/certs/localhost.pem


* verify if it's trusted using


        openssl verify localhost.crt


Unfortunately this does not work:


dotnet dev-certs https generates certificates that are affected by the issue described on https://github.com/openssl/openssl/issues/1418 and https://github.com/dotnet/aspnetcore/issues/7246:


    $ openssl verify localhost.crt
    CN = localhost
    error 20 at 0 depth lookup: unable to get local issuer certificate
    error localhost.crt: verification failed
    due to that it's impossible to have a dotnet client trust the certificate
    Workaround: (tested on Openssl 1.1.1c)


manually generate self-signed cert


trust this cert


force your application to use this cert


In detail:


manually generate self-signed cert:


create localhost.conf file with the following content:

    [req]
    default_bits       = 2048
    default_keyfile    = localhost.key
    distinguished_name = req_distinguished_name
    req_extensions     = req_ext
    x509_extensions    = v3_ca

    [req_distinguished_name]
    commonName                  = Common Name (e.g. server FQDN or YOUR name)
    commonName_default          = localhost
    commonName_max              = 64

    [req_ext]
    subjectAltName = @alt_names

    [v3_ca]
    subjectAltName = @alt_names
    basicConstraints = critical, CA:false
    keyUsage = keyCertSign, cRLSign, digitalSignature,keyEncipherment

    [alt_names]
    DNS.1   = localhost
    DNS.2   = 127.0.0.1


generate cert using


    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout localhost.key -out localhost.crt -config localhost.conf


convert cert to pfx using


    openssl pkcs12 -export -out localhost.pfx -inkey localhost.key -in localhost.crt


(optionally) verify cert using


    openssl verify -CAfile localhost.crt localhost.crt which


should yield


    localhost.crt: OK


as it's not trusted yet using openssl verify localhost.crt should fail with


    CN = localhost
    error 18 at 0 depth lookup: self signed certificate
    error localhost.crt: verification failed


trust this cert:


* copy localhost.crt to /usr/local/share/ca-certificates


        sudo cp -v localhost.crt /usr/local/share/ca-certificates/localhost.crt


* trust the certificate using


        sudo update-ca-certificates


* verify if the cert is copied to /etc/ssl/certs/localhost.pem (extension changes)


        ls /etc/ssl/certs/localhost.pem


* verifying the cert without the CAfile option should work now


        $ openssl verify localhost.crt
        localhost.crt: OK


force your application to use this cert


update your appsettings.json with the following settings:


    "Kestrel": {
      "Certificates": {
        "Default": {
          "Path": "localhost.pfx",
          "Password": ""
        }
      }
    }


Or:


    "Kestrel": {
      "Endpoints": {
        "HTTPS": {
          "Url": "https://localhost:5001",
          "Certificate": {
            "Path": "/etc/ssl/certs/<certificate.pfx>",
            "Password": "xyz123"
          }
        }
      }
    }

maybe the user need permissions for give ssl certifications


    sudo usermod -aG ssl-cert www-data


^ for usermod make changes, the user need, restart unix session, or only restar.


### For Chrome:


first install `certutil` command:


    sudo apt install libnss3-tools


now go to chrome for donwload the cert from url downloader tool:


* Click "Not Secure" in address bar.
* Click Certificate.
* Click Details.
* Click Export.


now Run:


        certutil -d sql:$HOME/.pki/nssdb -A -t "P,," -n {FILE_NAME} -i {FILE_NAME}


now restart google-chrome
