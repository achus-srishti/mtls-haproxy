## Certificate Authorities
When a client device needs to make an API call, it first establishes a secure connection to the HAProxy load balancer that’s situated in front of your servers. While making that connection, the client provides its certificate. To validate it, HAProxy checks whether it was digitally signed previously by your organization’s certificate authority.  
A certificate authority (CA) is a trusted entity that can vouch for the identity of another entity. In this case of client certificates and mTLS, your organization will use its own CA to sign all client certificates that it will deploy to applications. In essence, your organization becomes its own authority, doling out certificates and then verifying them when presented back to it.  
To demonstrate, we’ll create a root certificate authority for our organization. This root CA will sit at the top of our hierarchy of certificates, and as such, it would cause widespread problems if it were ever to become compromised. Therefore, we’ll also create an intermediate CA and then use the intermediate CA to sign client certificates.  

### Creating root CA
openssl req \  
    -newkey rsa:2048 \  
    -nodes \  
    -x509 \  
    -days 3650 \  
    -keyout ./certs/root-ca.key \  
    -out ./certs/root-ca.crt  

When run, the command prompts you to enter some additional information about your organization. One crucial question it asks is the name to assign to the CA. The **Common Name (CN) should be independent of any server name** and could be something like acme-root-ca. The command will **create two files, root-ca.key and root-ca.crt**.

The certificate file holds information about the CA, such as its name, expiration date, and issuer. Meanwhile, the key file contains the cryptographic bits associated with the certificate, which you can use to prove that you are the sole owner of the certificate and key. Keep the key private within your organization.

#### You can inspect the contents of the root-ca.crt file with the following command:
$ openssl x509 -noout -text -in ./certs/root-ca.crt  

Certificate:  
    Data:  
        Version: 3 (0x2)  
        Serial Number:  
            10:4d:04:cf:6e:42:4a:87:dc:6f:5d:54:c6:f6:cd:db:70:92:28:47  
        Signature Algorithm: sha256WithRSAEncryption  
        Issuer: C = US, ST = OH, L = Columbus, O = Acme, CN = acme-root-ca, emailAddress = admin@acme.com  
        Validity  
            Not Before: Aug  4 16:40:01 2022 GMT  
            Not After : Aug  1 16:40:01 2032 GMT  
        Subject: C = US, ST = OH, L = Columbus, O = Acme, CN = acme-root-ca, emailAddress = admin@acme.com  



### Create an Intermediate CA

Next, let’s create a CA that’s subordinate to the root CA. You can use this intermediate CA to sign client certificates. Use the following command to create its key and certificate signing request:

$ openssl req \  
   -newkey rsa:2048 \  
   -nodes \  
   -days 3650 \  
   -keyout ./certs/intermediate-ca.key \  
   -out ./certs/intermediate-ca.csr  

After asking you the same questions about your organization—for which you should choose a different Common Name—this **creates two files, intermediate-ca.key and intermediate-ca.csr**. **The first file is this CA’s private key and the second is a certificate signing request, which you will sign with the root CA key.**

Because the **root CA was self-signed, its type was automatically configured as a CA certificate**, as opposed to being designated an SSL server certificate or client certificate. With this new, **intermediate CA**, we’ll need to **state explicitly** that we want **it to be a CA type**, which we do by defining an extensions file. First, create a file named **ca-cert-extensions.cnf** and add extra information to append to the certificate. It should look like this:

basicConstraints = CA:TRUE  
keyUsage = keyCertSign, cRLSign  
subjectKeyIdentifier = hash  
authorityKeyIdentifier = keyid,issuer  


#### Then use the following openssl command to sign the certificate signing request and create the certificate:
$ openssl x509 \  
   -req \  
   -in ./certs/intermediate-ca.csr \  
   -out ./certs/intermediate-ca.crt \  
   -CA ./certs/root-ca.crt \  
   -CAkey ./certs/root-ca.key \  
   -CAcreateserial \  
   -days 3650 \  
   -extfile ./certs/ca-cert-extensions.cnf  

### Create a Client Certificate

Now that we have an intermediate certificate, the next step is to create a client certificate request and then sign it to create a certificate. For this example, assume that the **certificate will be installed into a “scanner” application running in a warehouse**. Create the certificate signing request with the following command:

$ openssl req \  
   -newkey rsa:2048 \  
   -nodes \  
   -days 365 \  
   -keyout ./certs/client.key \  
   -out ./certs/client.csr  

*Leverage this during automation: -subj "/CN=scanner/O=warehouse"*  

Next, let’s create an extensions file to ***designate this certificate as a client certificate***. Add a file named ***client-cert-extensions.cnf*** with the following contents:  

basicConstraints = CA:FALSE  
keyUsage = digitalSignature  
extendedKeyUsage = clientAuth  
subjectKeyIdentifier = hash  
authorityKeyIdentifier = keyid,issuer  

#### Then use the following command to sign the certificate signing request:

$ openssl x509 \  
   -req \  
   -in ./certs/client.csr \  
   -out ./certs/client.crt \  
   -CA ./certs/intermediate-ca.crt \  
   -CAkey ./certs/intermediate-ca.key \  
   -CAcreateserial \  
   -days 365 \  
   -extfile ./certs/client-cert-extensions.cnf  

*HAProxy need both cert and private key in haproxy cfg.*
cat ./certs/client.key ./certs/client.crt > ./certs/client.pem  

You now have a client certificate that you can use to authenticate our fictitious scanner application and gain access to an API.  

### Create a self signed certificate to start HAProxy server at port 443
openssl req -x509 -newkey rsa:4096 -keyout ./certs/server.key -out ./certs/server.crt -sha256 -days 3650 -nodes -subj "/C=IN/ST=KL/L=COK/O=uhca/OU=legends/CN=ca-legends-server-ca"  

HAProxy needs both to be in same file, if not haproxy looks for certificate file + ".key" as private key in the same location as cert file.
*cat ./certs/server.key ./certs/server.crt > ./certs/server.pem*  

### Configure haproxy to start at port 443 for client (masters) server 

frontend my_ssl  
    bind localhost:80  
    bind localhost:443 ssl crt ./certs/server.pem  
    default_backend web_servers  
backend web_servers  
    server server1 localhost:444 ssl verify none crt ./certs/client.pem  

ssl verify none is to ensure that server we are forwarding we are not validating server cert, this is becaise server cert is self signed  



### Configure haproxy to start at port 443 for backend (legends) server 

frontend my_ssl  
    bind localhost:81  
    bind localhost:444 ssl crt ./certs/server.pem ca-file ./certs/intermediate-ca.crt ca-verify ./certs/root-ca.crt  
    default_backend web_servers  
backend web_servers  
    server server1 localhost:8080  


### Errata:
https://stackoverflow.com/questions/10175812/how-to-generate-a-self-signed-ssl-certificate-using-openssl  
https://www.haproxy.com/blog/restrict-api-access-with-client-certificates-mtls  
https://www.haproxy.com/blog/haproxy-ssl-termination  