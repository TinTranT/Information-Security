# Task 1: Transfer files between computers  
**Question 1**: 
Conduct transfering a single plaintext file between 2 computers, 
Using openssl to implementing measures manually to ensure file integerity and authenticity at sending side, 
then veryfing at receiving side. 

**Answer 1**:

***Step 1: Set up environment***

We have 2 linux user

User1 have ip 
User2 have ip 

Set up ssh for both user

```sh
sudo apt update
sudo apt install openssh-server
sudo systemctl start ssh
```

***Step 2: Create sample file in user1***

```sh
echo "This is a sample text from user1" > file.txt
```

Now we watch the file we have just created

```sh
cat file.txt
```

***Step 3: Ping 2 user to ensure they are connected***

***Step 4: Transfering a plaintext file between 2 user***

```sh
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -pubout -out public_key.pem
```

Calculate the hash of the file

```sh
openssl dgst -sha256 -out file.txt.sha256 file.txt
```

Create digital signature with private key

```sh
openssl dgst -sha256 -sign private_key.pem -out file.txt.sign file.txt
```

Transfer files to user2 with scp

```sh
scp file.txt file.txt.sha256 file.txt.sign public_key.pem ubuntu@10.111.5.171:/home/user
```

Verify from user2

```sh
openssl dgst -sha256 -verify public_key.pem -signature file.txt.sign file.txt
```

# Task 2: Transfering encrypted file and decrypt it with hybrid encryption. 
**Question 1**:
Conduct transfering a file (deliberately choosen by you) between 2 computers. 
The file is symmetrically encrypted/decrypted by exchanging secret key which is encrypted using RSA. 
All steps are made manually with openssl at the terminal of each computer.

**Answer 1**:


# Task 3: Firewall configuration
**Question 1**:
From VMs of previous tasks, install iptables and configure one of the 2 VMs as a web and ssh server. Demonstrate your ability to block/unblock http, icmp, ssh requests from the other host.

**Answer 1**:
***Step 1: Update docker-compose.yml***
Configure inner-172.16.10.100 as the firewall to block/unblock HTTP, ICMP, and SSH requests.
```sh
inner:
        build: 
            context: ./image
        image: base-image
        container_name: inner-172.16.10.100
        tty: true
        cap_add:
            - ALL
        networks:
            net-172.16.10.0:
                ipv4_address: 172.16.10.100
        command: bash -c "
                      ip route del default &&
                      ip route add default via 172.16.10.10 &&
                      /etc/init.d/openbsd-inetd start &&
                      tail -f /dev/null
                 "
```
Ensure iweb-172.16.10.110 is set up as the web and SSH server.
```sh
apache1:
        build: 
            context: ./apache
        image: apache-image
        container_name: iweb-172.16.10.110
        tty: true
        cap_add:
            - ALL
        networks:
            net-172.16.10.0:
                ipv4_address: 172.16.10.110
        command: bash -c "
                      ip route del default &&
                      ip route add default via 172.16.10.10 &&
                      service ssh start &&
                      service apache2 start &&
                      tail -f /dev/null
                "
```
***Step 2: Build and run the Docker containers***
Build the Docker images and start the containers using docker-compose.
```sh
docker-compose up -d
```
***Step 3: Demonstrate blocking/unblocking HTTP, ICMP, and SSH requests***

Access the inner-172.16.10.100 container to add/remove iptables rules.

```sh
docker exec -it inner-172.16.10.100 /bin/bash
```

Then we edit iptables:

1. Block HTTP requests (port 80).

```sh
iptables -A INPUT -p tcp --dport 80 -j REJECT
```

2. Block ICMP requests (ping).

```sh
iptables -A INPUT -p icmp -j REJECT
```

3. Block SSH requests (port 22).

```sh
iptables -A INPUT -p tcp --dport 22 -j REJECT
```

4. Unblock HTTP requests (port 80).

```sh
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

5. Unblock ICMP requests (ping).

```sh
iptables -A INPUT -p icmp -j ACCEPT
```

6. Unblock SSH requests (port 22).

```sh
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

