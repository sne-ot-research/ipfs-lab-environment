## Instructions

This docker-compose file helps in setting up an IPFS private topology using the IPFS docker image published [here](https://hub.docker.com/repository/docker/dadepo/ipfs-private). 

The image contains modified version of the `go-ipfs` codebase that ensures that a private network is always started. 
The source of the modified version can be found [here](https://github.com/dadepo/go-ipfs/tree/private-from-master)


### Starting up the IPFS network

Clone, or copy the content of this `docker-compose.yml` file. 

Create a directory at `/tmp/ipfs/nexus`, this is used as volume to share content across the containers.

:exclamation:
**NOTE: Ideally the docker container should take care of creating this directory, but for some reason I have not investigated yet, this did not happen on the server. I had to create it and give it a permission of 777.
TODO: Investigate this and resolve
**
:exclamation:

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

## Experiment Scenarios
-

### Testing blacklist functionality

#### On adding content

1.  Start services defined in docker compose by running: 
   `docker-compose up` in a terminal.

2.  In another terminal, enter into any of the containers. In this
    instruction, we use `node1`. Do this by running `docker exec -it
    node1 /bin/sh`

3.  Check that `blacklist` and `virus.jpg` file exist in the root directory

4.  Now attempt to add `virus.jpg` by running `ipfs add virus.jpg`

5.  You should see an error saying content is potentially malicious


#### On requesting content

1.  Start services defined in docker compose by running `docker-compose
    up` in a terminal.

2.  Enter into any of the containers. In this instruction, we use
    `node1`. Do this by running `docker exec -it node1 /bin/sh`

3.  Check that `blacklist` and `virus.jpg` file exist in root directory

4.  Edit `/blackist` file and remove the content. This is to allow the
    addition of `virus.jpg`

5.  Run `ipfs add virus.jpg` to add the `virus.jpg` to IPFS. Note the
    CID that is outputted.

6.  Exit `node1` and enter `node2` container

7.  Now attempt to fetch the CID gotten from output in previous step by
    running `ipfs get -CID from previous step-`

8.  Content will not be fetched and you should see an error saying content is potentially malicious


### Exploiting CORS for content injection and deletion

#### Injecting content

1.  Start services defined in docker compose by running `docker-compose
    up` in a terminal.

2.  In another terminal, enter into `open\_config\_node3` by running
    `docker exec -it open\_config\_node3 /bin/sh`

3.  Now run `ipfs cat QmSv6nkqA5ghKeQNoVxUZgnVhaRbirPDZgF5Qyf2RvMDtV`.
    Confirm that this command does not return. This shows no content with
    CID `QmSv6nkqA5ghKeQNoVxUZgnVhaRbirPDZgF5Qyf2RvMDtV` exist

4.  Now open a browser and access
    `http://localhost:8083/ipfs/QmPesj1PHfxPah9B5xV28kWbWwLbcFZD3ip3Sszy8uWFpm*`
    The CID *QmPesj1PHfxPah9B5xV28kWbWwLbcFZD3ip3Sszy8uWFpm* is for content added by
    `mal\_node4` that exploits the CORS configuration of
    `open\_config\_node3` to add content to it

5.  Now go back to `open\_config\_node3` and run `ipfs cat
    QmSv6nkqA5ghKeQNoVxUZgnVhaRbirPDZgF5Qyf2RvMDtV` this time the
    command returns showing that content has been added to it.


#### Deleting content

1.  Start services defined in docker compose by running *docker-compose
    up* in a terminal.

2.  Connect to `mal\_node4` by running `docker exec -it mal\_node4
    /bin/sh`

3.  Enabling listening to DHT communication by running `ipfs log level
    dht debug`

4.  In another terminal, now connect to `open\_config\_node3` by running
    `docker exec -it open\_config\_node3 /bin/sh`

5.  Add content to *open\_config\_node3*. For example:
   `echo "hello world" \> data.txt && ipfs add data.txt`

6.  Confirm content was added to IPFS by running:
    `ipfs cat CID\_FROM\_PREVIOUS\_STEP`

7.  Go back to `mal\_node4` and run `peertheripper /tmp/stderr`

8.  Return to `open\_config\_node3`, and confirm that the content that
    was added has been deleted. Do this by running: 
    `ipfs cat CID\_FROM\_PREVIOUS\_STEP`. This time around, the command won’t
    return any content and after a while will timeout

### Viewing Pubsub log entries

1.  Start services defined in docker compose by running: 
    `docker-compose up` in a terminal.

2.  Connect to `pubsub\_1\_node5` by running `docker exec -it
    pubsub\_1\_node5 /bin/sh`

3.  Connect to `pubsub\_1\_node5` from another terminal in other to tail
    the log. Once connected to `pubsub\_1\_node5`, run `tail -f
    /tmp/stderr`

4.  Switch log level of pubsub component to debug by running `ipfs log
    level pubsub debug`

5.  Subscribe to a topic. For instance `pubsub-topic` by running `ipfs
    pubsub sub pubsub-topic`

6.  Unsubscribe by pressing `Ctr+C`

7.  In the other terminal tailing the logs, there should be entries that
    shows that the peer joined and left the `pubsub-topic` topic

