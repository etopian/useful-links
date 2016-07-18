copied from: http://www.hydrogen18.com/blog/your-own-pki-tls-golang.html

This copy exists only for archive purposes.

I recently decided to tackle the problem of securing communciation between two processes written in Go. The logical answer is to use the crypto/tls implementation already available in Golang. The typical use case of TLS is to have the client verify the identity of the server. This is the most common usage of TLS when it is used as part of HTTPS. In my case I needed to have the server verify the clients identity as well.

Since I am not trying to authenticate a third party, it is much easier to create my own Certificate Authority in this circumstance. A Certificate Authority signs certificates that allow entities to prove their identity to one another. This only works if both entities trust the Certificate Authority. Most operating systems include a relatively large "bundle" of various Certificate Authorities by default.

I quickly ran into an issue however. Client authentication of certificates is part of the "version 3" X.509 specification. I spent about an hour muddling around with various options to the openssl tool and got nowere. Surely there is a better way!

It turns out this same use case is part of the OpenVPN project. So much that an offshoot of OpenVPN is a project called EasyRSA. Setting up a certificate authority and signing client certificates is relatively easy with this tool.
Creating the certificates

To verify a connection between a client and a server here is the steps we need to take at a high level.

    Create a Certificate Authority. This is commonly called a "CA".
    Distribute the root certificate to all clients and servers.
    Generate a server certificate for the server.
    Use the CA to sign the server certificate.
    Generate a client certificate for the client.
    Use the CA to sign the client certificate.
    Configure the server to trust the CA to authenticate clients.
    Configure the client to trust the CA to authenticate servers.

Using EasyRSA

EasyRSA is designed to be cloned and used in place. First lets clone the repository from github.

ericu@eric-phenom-linux:~$ git clone https://github.com/OpenVPN/easy-rsa.git example-ca
Cloning into 'example-ca'...
remote: Reusing existing pack: 323, done.
remote: Total 323 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (323/323), 122.61 KiB | 0 bytes/s, done.
Resolving deltas: 100% (124/124), done.
Checking connectivity... done.
ericu@eric-phenom-linux:~$ chmod 700 example-ca
ericu@eric-phenom-linux:~$ cd example-ca/
ericu@eric-phenom-linux:~/example-ca$ rm -rf .git

First I cloned the repository into a folder called example-ca that I'll be using for the purposes of this article. Next I changed the permissions so that only I could access the directory. Then I deleted the internal .git folder used by the git revision control system. There is no further need for it.
Certificate Authority Setup

Next we need to setup the CA.

ericu@eric-phenom-linux:~/example-ca/easyrsa3$ ./easyrsa init-pki

init-pki complete; you may now create a CA or requests.
Your newly created PKI dir is: /home/ericu/example-ca/easyrsa3/pki

ericu@eric-phenom-linux:~/example-ca/easyrsa3$ ./easyrsa build-ca
Generating a 2048 bit RSA private key
..............+++
..................+++
writing new private key to '/home/ericu/example-ca/easyrsa3/pki/private/ca.key'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:"Example CA"

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/home/ericu/example-ca/easyrsa3/pki/ca.crt

ericu@eric-phenom-linux:~/example-ca/easyrsa3$ 

Afte changing into the easyrsa3 directory you can run the easyrsa script. All actions are performed using this script. The first step is to run the init-pki command. Then you run the build-ca command. Both of these steps only need to be ran once no matter how many certificates you need to create for clients and servers.

The first prompt you'll get is for a password for the Certificate Authority. This password is used when signing any future certificates for clients and servers. It's possible to not use a password, but I highly reccomend against it. Next you'll be prompted for a name of the CA, pick something meaningful to you.

At this point EasyRSAs has created a number of files. You should always kept the directory secret. You do however need to distribute the root certificate for your CA. This file is at the location pki/ca.crt relative to the current directory. It's safe to distribute this file to anyone.
Server certificate

Now we can create a certificate for the server. In this example I'm going to be using a client and server communicating on the loopback interface so I am going to sign a certificate for the name "localhost". You can follow along here and generate a certificate for "localhost" and use it with my example code later on. This will confirm that everything is working.

In the real world you'll need to sign a certificate using the domain name of the server.

ericu@eric-phenom-linux:~/example-ca/easyrsa3$ ./easyrsa build-server-full localhost nopass
Generating a 2048 bit RSA private key
.........+++
.......................................+++
writing new private key to '/home/ericu/example-ca/easyrsa3/pki/private/localhost.key'
-----
Using configuration from /home/ericu/example-ca/easyrsa3/openssl-1.0.cnf
Enter pass phrase for /home/ericu/example-ca/easyrsa3/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :PRINTABLE:'localhost'
Certificate is to be certified until Jun 26 00:47:56 2024 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated

This step only requires a single command. You'll be prompted to enter the password you used when you setup the CA again. The name of the server is passed in on the command line. The nopass option is passed here. It is generally undesireable to place a password on a server certificate because it means the server needs human intervention to start.

EasyRSA generates several more files as part of this step. They are pki/issued/localhost.crt and pki/private/localhost.key. The server needs both of these files to prove its identity to clients. The localhost.key file constitutes the secret portion of the key and should under no circumstances be distributed.

If you had signed a certificate instead for helpdesk.internal.bigcorp.com EasyRSA would have generated the files pki/issued/helpdesk.internal.bigcorp.com.crt and pki/private/helpdesk.internal.bigcorp.com.key.
Client Certificate

Generation of the client certificate is almost identical to that of the server certificate.

ericu@eric-phenom-linux:~/example-ca/easyrsa3$ ./easyrsa build-client-full 'client0' nopass
Generating a 2048 bit RSA private key
.+++
.............................................+++
writing new private key to '/home/ericu/example-ca/easyrsa3/pki/private/client0.key'
-----
Using configuration from /home/ericu/example-ca/easyrsa3/openssl-1.0.cnf
Enter pass phrase for /home/ericu/example-ca/easyrsa3/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :PRINTABLE:'client0'
Certificate is to be certified until Jun 26 00:59:27 2024 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
ericu@eric-phenom-linux:~/example-ca/easyrsa3$ 

This again generates two files pki/issued/client0.crt and pki/private/client0.key. The client needs both files to prove its identity to the server. The name given to the client is less important than the name signed on the server's certificate. Servers generally do not attempt to verify the name of a client, only that its certificate was signed by a trusted CA.
A note on encodings

There are many different encodings which can be used to store the certificates and keys. I'm not elaborating on them here because EasyRSA generates them in formats that are suitable for use in Go by default. If you're using a different language you may need support libraries to read them. Alternatively you can use the openssl tool to convert the files into other encoding formats.
Putting it together with Go

Let's dive into the crypto/tls package in Go.

Fundamentally the server needs to use the tls.Listen function to listen for incoming TLS connections. The client needs to establish a connection to the server using tls.Dial.

By default, the server will not verify the client's certificate. The client nor the server know to trust the CA because it has not been added to the operating systems bundle.

The behavior of the TLS client and server in Go are governed by the tls.Config structure. It is passed in when using the functions in the crypto/tls package. This structure is quite large and complex because it is based on the nightmarish X.509 standard.
Trusting the new CA

The RootCAs member of the configuration is normally nil. In this case, Go uses the host's certificates. To use our CA we need to add it an instance of x509.CertPool and assign RootCAs to point at that instance. This is quite easy. A new instance is created using x509.NewCertPool() from the crypto/x509 package. The contents of the ca.crt file can be read and passed directly to x509.AppendCertsFromPEM. This function returns true if everything was loaded successfully.
Verifying clients

To get the server to verify clients the ClientAuth member of the tls.Config structure needs to be changed to tls.RequireAndVerifyClientCert. However, the server code attempts to verify against a different pool of certificate authorities than used by the client code. This is contained in the ClientCAs member. Since we have already configured our CA in the previous step, we can assign ClientCAs to point at the same instance of x509.CertPool as RootCAs.
Verifying the server

By default the client attempts to verify servers. There is nothing to do here.
Loading the certficiate and private key

Both the server and client need to load their certificate and private key. This can be done in a single step using the tls.LoadX509KeyPair() function to load directly from the two seperate files. They second parameter is the .key file in both circumstances. If you have already have the files read into memory you can use the tls.X509KeyPair().

The resulting x509.Certificate needs to be placed in the Certificates member of the tls.Config structure. The configuration allows for more than one certificate by using a slice. So in this case, just create a slice of length one and assign it to the single x509.Certificate.
Example Code

This wouldn't be complete without example code. You can get the code on github.

Once you've got your GOPATH environmental variable set up it is easy to get it running. There is an executable for both a client and a server. Each executable expects the following arguments in order

    The private key file.
    The certificate key file for itself.
    The root certificate of the CA.

Here is how to run the server from a bash terminal.

ericu@eric-phenom-linux:~/liteide$ go get github.com/hydrogen18/test-tls
ericu@eric-phenom-linux:~/liteide$ go install github.com/hydrogen18/test-tls/server
ericu@eric-phenom-linux:~/liteide$ $GOPATH/bin/server ~/example-ca/easyrsa3/pki/private/localhost.key ~/example-ca/easyrsa3/pki/issued/localhost.crt ~/example-ca/easyrsa3/pki/ca.crt

You'll need to change the paths to the files to wherever you setup your CA.

To run the client is almost the same.

ericu@eric-phenom-linux:~/liteide$ go get github.com/hydrogen18/test-tls
ericu@eric-phenom-linux:~/liteide$ go install github.com/hydrogen18/test-tls/client
ericu@eric-phenom-linux:~/liteide$ $GOPATH/bin/client ~/example-ca/easyrsa3/pki/private/client0.key ~/example-ca/easyrsa3/pki/issued/client0.crt ~/example-ca/easyrsa3/pki/ca.crt
Hello TLS

Once again, change the paths to the appropriate location on your system. If everything works you should see the "Hello TLS" message. My hope is this example code can be used as a guide to implement your own software.
Going further

Since both of my clients are using the same implementation, there should be no problem restricting the options for communications between the client and server. Typically HTTPS servers have to support a large number of ciphers and other options due to the wide the number of clients.
Restricting the ciphers in use

By default, the crypto/tls package supports all ciphers including those based off the older Triple DES standard. We can restrict this signficiantly by setting the CipherSuites member of the tls.Config structure.

    //Use only modern ciphers
    config.CipherSuites = []uint16{tls.TLS_RSA_WITH_AES_128_CBC_SHA,
        tls.TLS_RSA_WITH_AES_256_CBC_SHA,
        tls.TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,
        tls.TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA,
        tls.TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,
        tls.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,
        tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256}

Only use TLS v1.2

There is no reason use older versions of TLS/SSL. Set the MinVersion member of the tls.Config structure to tls.VersionTLS12 use only TLS version 1.2
Disable session tickets

Session tickets are only needed if you want to support session resumption. I have no need for this. Set the SessionTicketsDisabled member of the tls.Config structure to true to disable this
