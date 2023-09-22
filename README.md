# mongodb_replica
# Using haproxy to healthcheck for mongodb cluster with param:

  global
      log /dev/log local0
      log /dev/log local1 notice
      user haproxy
      group haproxy
      daemon
      maxconn 4096

  defaults
      log global
      mode tcp
      option tcplog
      option dontlognull
      timeout connect 5000
      timeout client 50000
      timeout server 50000
  frontend stats
      mode http
      bind *:1024
      stats enable
      stats uri /stats
      stats refresh 10s
      stats auth admin:admin

  frontend mongodb_frontend
      bind *:27017
      mode tcp
      option tcplog
      default_backend mongodb_backend

  backend mongodb_backend
      option tcp-check

      # Credit: https://blog.danman.eu/mongodb-haproxy/
      # Wire protocol
      tcp-check send-binary 3a000000 # Message Length (58)
      tcp-check send-binary EEEEEEEE # Request ID (random value)
      tcp-check send-binary 00000000 # Response To (nothing)
      tcp-check send-binary d4070000 # OpCode (Query)
      tcp-check send-binary 00000000 # Query Flags
      tcp-check send-binary 61646d696e2e # fullCollectionName (admin.$cmd)
      tcp-check send-binary 24636d6400 # continued
      tcp-check send-binary 00000000 # NumToSkip
      tcp-check send-binary FFFFFFFF # NumToReturn
      # Start of Document
      tcp-check send-binary 13000000 # Document Length (19)
      tcp-check send-binary 10 # Type (Int32)
      tcp-check send-binary 69736d617374657200 # ismaster:
      tcp-check send-binary 01000000 # Value : 1
      tcp-check send-binary 00 # Term
      tcp-check expect binary 69736d61737465720001 #ismaster True
      
      # Severs. Note that the port is the internal port of the MongoDB container.
      server mongo1 mongodb-0-external:27017 check inter 3s fall 3 rise 2
      server mongo2 mongodb-1-external:27017 check inter 3s fall 3 rise 2

# Service of haproxy listen all port of mongo:
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http
  - name: https
    protocol: TCP
    port: 443
    targetPort: https
  - name: mongo
    protocol: TCP
    port: 27017
    targetPort: mongo
    nodePort: 32217
  - name: stat
    protocol: TCP
    port: 1024
    targetPort: stat

# delete pod for testing
# inbound traffic forward to haproxy and reroute to mongo pod

![image](https://github.com/luutrungnguyen/mongodb_replica/assets/89311737/b74f5c51-c2df-4e13-9829-24bba05bd1c6)

