# Crypto using spiped, Secure Pipe (daemon)

加密技術原理

The idea of adding SSL(secure socket layer) support to Redis was proposed many times, however currently we believe that given the small percentage of users requiring SSL support, and the fact that each scenario tends to be different, using a different "tunneling" strategy can be better.

Spiped is a utility for creating symmetrically encrypted and authenticated pipes between socket addresses, so that one may connect to one address (e.g., a UNIX socket on localhost) and transparently have a connection established to another address (e.g., a UNIX socket on a different system).

# Abatract, 抽象概念

Spiped (pronounced "ess-pipe-dee") is a utility for creating symmetrically encrypted and authenticated pipes between socket 通訊端 addresses, so that one may connect to one address (e.g., a UNIX socket on localhost) and transparently have a connection established to another address (e.g., a UNIX socket on a different system). This is similar to 'ssh -L' functionality, but does not use SSH and requires a pre-shared symmetric key.

# Spiped Cryptographic components, 使用的加密元件

The initial key negotiation is performed using HMAC-SHA256 and an authenticated Diffie-Hellman key exchange over the standard 2048-bit "group 14"; following the completion of key negotiation, packets are transmitted encrypted with AES-256 in CTR mode and authenticated using HMAC-SHA256. The simplicity of the code — about 6000 lines of C code in total, of which under 2000 are specific to spiped (the rest is library code originating from kivaloo and Tarsnap) — makes it unlikely that spiped has any security vulnerabilities.

On the author's 2.5 GHz Intel Core 2 laptop, spiped operates at approximately 300 Mbps.

# Working Flow of Mail Transfer using port 25, 加密後的郵件傳輸

before
                    
                                                    0.0.0.0:25
                  Client   ---------------------->   Server
                                     SMTP      firewall


after spiped

                                                     127.0.0.1:25
                   Client   ---------------------->   Server     
           spiped 127.0.0.1:25      SPIPED     firewall    
                                                 spiped 0.0.0.0:8025
                                (wrapped smtp)
                                
# Working Flow of SSH Transfer using port 22  

SSH before
                    
                                                    0.0.0.0:25
                  Client   ---------------------->   Server
                                     SMTP      firewall
                                     
                                     
SSH after spiped

                                                     127.0.0.1:22
                  Client   ---------------------->   Server     
                                   SPIPED     firewall    
                                                 spiped 0.0.0.0:8022
                                (wrapped ssh)         
                              

# Principle, 工作原理

To set up an encrypted and authenticated pipe for sending email between two systems (in this case, from many systems around the internet to his central SMTP server, which then relays email to the rest of the world), one might run

    dd if=/dev/urandom bs=32 count=1 of=keyfile
    spiped -d -s '[0.0.0.0]:8025' -t '[127.0.0.1]:25' -k keyfile
    
 on a server and after copying keyfile to the local system, run
 
    spiped -e -s '[127.0.0.1]:25' -t $SERVERNAME:8025 -k keyfile
    
at which point mail delivered via localhost:25 on the local system will be securely transmitted to port 25 on the server (which is configured to relay mail which arrives from 127.0.0.1 but not from other addresses).

You can also use spiped to protect SSH servers from attackers: Since data is authenticated before being forwarded to the target, this can allow you to SSH to a host while protecting you in the event that someone finds an exploitable bug in the SSH daemon -- this serves the same purpose as port knocking or a firewall which restricts source IP addresses which can connect to SSH. On the SSH server, run

    dd if=/dev/urandom bs=32 count=1 of=/etc/ssh/spiped.key
    spiped -d -s '[0.0.0.0]:8022' -t '[127.0.0.1]:22' -k /etc/ssh/spiped.key
    
 then copy the server's 
 
     /etc/ssh/spiped.key
     
 to below path on your local system 
 
    ~/.ssh/spiped_HOSTNAME_key 
    
 and line below lines to to the ~/.ssh/config file.
 
    Host HOSTNAME
    ProxyCommand spipe -t %h:8022 -k ~/.ssh/spiped_%h_key
   
This will cause ssh HOSTNAME to automatically connect using the spipe client via the spiped daemon; you can then firewall off all incoming traffic on port tcp/22.

# Versioning till 2017

 seems it is replaced by the Asymmetric Encryption since Spiped is Symmetric Encryption Tech.
 
# Links

http://www.tarsnap.com/spiped.html
 
 
 
 
 
 
 
 

