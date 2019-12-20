# redisSecurity

此篇技術文稍有技術門檻，讀者需要有資料庫和網路基礎。

# platforms Mediate, 斡旋於各平台間的仲介軟體

Redis is designed to be accessed by trusted clients inside trusted environments. This means that usually it is not a good idea to expose the Redis instance directly to the internet or, in general, to an environment where untrusted clients can directly access the Redis TCP port or UNIX socket.

For instance, in the common context of a web application implemented using Redis as a database, cache, or messaging system, the clients inside the front-end (web side) of the application will query Redis to generate pages or to perform operations requested or triggered by the web application user.

In this case, the web application mediates access between Redis and untrusted clients (the user browsers accessing the web application).


                                        Redis Intance   Redis Server
        user || untrust user -> browser -> Client App -> Server
                                                         Secure Model:
                                                          (1)ACL
                                                          (2)Validate input
                                                                                         

This is a specific example, but, in general, untrusted access to Redis should always be mediated by a layer implementing ACLs, validating user input, and deciding what operations to perform against the Redis instance.
In general, Redis is not optimized for maximum security but for maximum performance and simplicity.

# Security Model, 安全模組如下：

(1)網路: 通訊阜號 和 IP 位址

Access to the Redis port should be denied to everybody but trusted clients in the network, so the servers running Redis should be directly accessible only by the computers implementing the application using Redis.

In the common case of a single computer directly exposed to the internet, such as a virtualized Linux instance (Linode, EC2, ...), the Redis port should be firewalled to prevent access from the outside. Clients will still be able to access Redis using the loopback interface.

    Note that it is possible to bind Redis to a single interface by adding a line like the following to the redis.conf file:

    bind 127.0.0.1
    ...
    

Failing to protect the Redis port from the outside can have a big security impact because of the nature of Redis. For instance, a single FLUSHALL command can be used by an external attacker to delete the whole data set.

(2)登入驗證

When the authorization layer is enabled, Redis will refuse any query by unauthenticated clients. A client can authenticate itself by sending the AUTH command followed by the password.

The password is set by the system admin in clear text inside the redis.conf file. It should be long enough to prevent brute force attacks for two reasons:

    1.Redis is very fast at serving queries. Many passwords per second can be tested by an external client.

    2.The Redis password is stored inside the redis.conf file and inside the client configuration, so it does not need to be remembered by the system administrator, and thus it can be very long.
    ...
    

The goal of the authentication layer is to optionally provide a layer of redundancy. 

If firewalling or any other system implemented to protect Redis from external attackers fail, an external client will still not be able to access the Redis instance without knowledge of the authentication password.

The AUTH command, like every other Redis command, is sent unencrypted, so it does not protect against an attacker that has enough access to the network to perform eavesdropping. (客戶端應用程式管理層級的指令遭竊聽則不在安全保護範圍內。)

(3)加密資料

  https://github.com/QuinoaPy/Crypto
  
  The idea of adding SSL(secure socket layer) support to Redis was proposed many times, however currently we believe that given the small percentage of users requiring SSL support, and the fact that each scenario tends to be different, using a different "tunneling" strategy can be better.

Spiped is a utility for creating symmetrically encrypted and authenticated pipes between socket addresses, so that one may connect to one address (e.g., a UNIX socket on localhost) and transparently have a connection established to another address (e.g., a UNIX socket on a different system).

(4)指令重新命名（隱藏包覆指令）、 資料庫惡意注入、字串編碼

重新命名指令

      rename-command CONFIG b840fc02d524045
      rename-command CONFIG ""

資料庫惡意注入

    There is a class of attacks that an attacker can trigger from the outside even without external access to the instance. An example of such attacks are the ability to insert data into Redis that triggers pathological (worst case) algorithm complexity on data structures implemented inside Redis internals.

    For instance an attacker could supply, via a web form, a set of strings that is known to hash to the same bucket into a hash table in order to turn the O(1) expected time (the average time) to the O(N) worst case, consuming more CPU than expected, and ultimately causing a Denial of Service.

    To prevent this specific attack, Redis uses a per-execution pseudo-random seed to the hash function.
    Redis implements the SORT command using the qsort algorithm. Currently, the algorithm is not randomized, so it is possible to trigger a quadratic worst-case behavior by carefully selecting the right set of inputs.

    以上說明了如果惡意入侵者即便未取得存取權限，仍能由腳本惡意過多注入資料至內部資料庫，讓伺服器端的無物癱瘓。

    為了防止如此惡意行為， redis 擁有安全機制 per-execution pseudo-random seed 可以謹慎選擇外來的輸入，方才觸發其內部執行。
      ...
      

字串編碼解碼

    正因為 redis 並未支援客戶端常用套件如字串編解碼的函式庫，如 EVAL 或是 EVALSHA，所以入侵者即便執行此指令對 redis 來說沒有作用。
    ...

(5)限制配置代碼存放的目錄

In a classical Redis setup, clients are allowed full access to the command set, but accessing the instance should never result in the ability to control the system where Redis is running.

Internally, Redis uses all the well known practices for writing secure code, to prevent 

1.buffer overflows
2.format bugs 
3.memory corruption 

However, the ability to control the server config using the CONFIG command makes the client able to change the working dir of the program and the name of the dump file.

This allows clients to write RDB Redis files at random paths, that is a security issue that may easily lead to the ability to compromise the system and/or run untrusted code as the same user as Redis is running.

Redis does not requires root privileges to run. It is recommended to run it as an unprivileged redis user that is only used for this purpose. The Redis authors are currently investigating the possibility of adding a new configuration parameter to prevent CONFIG SET/GET dir and other similar run-time configuration directives. This would prevent clients from forcing the server to write Redis dump files at arbitrary locations.

(6)GPG加密技術：

   https://zh.wikibooks.org/zh-tw/GPG
   
   https://gnupg.org/download/


