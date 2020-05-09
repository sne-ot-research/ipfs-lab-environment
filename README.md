## Instructions

This docker-compose file helps in setting up an IPFS private topology using the IPFS docker image published [here](https://hub.docker.com/repository/docker/dadepo/ipfs-private). 

The image contains modified version of the `go-ipfs` codebase that ensures that a private network is always started. 
The source of the modified version can be found [here](https://github.com/dadepo/go-ipfs/tree/private-from-master)


### Starting up the IPFS network

Clone, or copy the content of this `docker-compose.yml` file. 

Create a directory at `/tmp/ipfs/nexus`, this is used as volume to share content across the containers.

**NOTE: Ideally the docker container should take care of creating this directory, but for some reason I have not investigated yet, this did not happen on the server. I had to create it and give it a permission of 777.
TODO: Investigate this and resolve
**


Then run

`docker-compose up` or `docker-compose up -d` to start the processes in the background.


### Testing the IPFS network

Run `docker ps` to see the running containers.

Connect to the bootnode (or any of the nodes) by running:

`docker exec -it bootnode /bin/sh`

To confirm the IPFS node is running as a private network run:

`ipfs bootstrap list` 

This should return nothing.

`ipfs swarm peers`

This should return only IP addresses configured for the nodes in the yaml file 

To test that files can be accessed within the network, connect to any of the nodes and add a file. For example on bootnode:

```
/ # echo "Hello IPFS!" > file1.txt
/ # ipfs add file1.txt
added QmYWAifyw2V5dEq7c5GgdSPffeKoYXQZggnYzw5RbXpig4 file1.txt
```

Then go to another node, and confirm you can retrieve the file added in the fist node using the hash. For exampple on node1:

```
ipfs get QmYWAifyw2V5dEq7c5GgdSPffeKoYXQZggnYzw5RbXpig4
/ # cat QmYWAifyw2V5dEq7c5GgdSPffeKoYXQZggnYzw5RbXpig4
Hello IPFS!
```

You can also retrieve the content via the HTTP gateway. That is:

```
http://localhost:8080/ipfs/<hash>

http://localhost:8080/ipfs/QmYWAifyw2V5dEq7c5GgdSPffeKoYXQZggnYzw5RbXpig4
```

### Adding nodes to the IPFS network

To add nodes to the network, edit the docker-compose.yml:

```
    nodeN: <-- change seervice name
      depends_on:
          - bootnode
      container_name: nodeN <--- change node name
      image: dadepo/ipfs-private:latest
      volumes:
          - /tmp/ipfs/nexus:/usr/local/nexus/
      expose:
          - 8080
      ports:
         - 808N:8080 <--- change port number 
      networks:
        private_net:
             ipv4_address: "192.168.1.10N" <-- change IP address
 ```

## Attack Scenarios
-

### Exploiting insecure COS configuration

After starting the containers, access `QmPesj1PHfxPah9B5xV28kWbWwLbcFZD3ip3Sszy8uWFpm` from the gateway of any of the nodes. For example:

```
http://localhost:8081/ipfs/QmPesj1PHfxPah9B5xV28kWbWwLbcFZD3ip3Sszy8uWFpm
```

The QmPesj1PHfxPah9B5xV28kWbWwLbcFZD3ip3Sszy8uWFpm is a malicious html page added by the `mal_node4` node which exploits `open_config_node3` node which has an insecure COS configuration

