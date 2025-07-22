
## Step to reproduce

## Swarm Cluster setup
master:   10.30.11.11
worker1:  10.30.11.12
worker2:  10.30.11.13

## Initialize Swarm Cluster
all node:

````
sudo apt remove docker docker-engine docker.io containerd runc
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce=5:28.2.2-1~ubuntu.22.04~jammy docker-ce-cli=5:28.2.2-1~ubuntu.22.04~jammy containerd.io

sudo systemctl restart docker && sudo systemctl enable docker
````

master:
````
sudo docker swarm init --advertise-addr <master ip>
````

worker1 & worker2:
````
sudo docker swarm join --token <join token> <master ip>:2377
````

## set all interface mtu to 9001 on all nodes (in case: ens19)
sudo ip link set dev ens19 mtu 9001

## disable all icmp for all (real case)
sudo iptables -A INPUT -p icmp -j DROP
sudo iptables -A OUTPUT -p icmp -j DROP

## Create sample app (on master node)
````
mkdir flask-app && cd flask-app

# app.py
cat <<EOF > app.py
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/send', methods=['POST'])
def receive_large_data():
    data = request.data
    print(f"Received data of size: {len(data)} bytes")
    return jsonify({"status": "received", "size": len(data)})

@app.route('/generate', methods=['GET'])
def generate_large_data():
    payload_size = int(request.args.get('size', 65535))
    payload = "xi" * payload_size
    return payload

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Dockerfile
cat <<EOF > Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY app.py .

RUN pip install flask

EXPOSE 5000

CMD ["python", "app.py"]
EOF

# docker-compose-worker1.yml
cat <<EOF > docker-compose1.yml
version: "3.8"

services:
  data-api-worker1:
    build: .
    image: vourteen14/data-api:latest
    ports:
      - target: 5000
        published: 8088
        protocol: tcp
        mode: ingress
    deploy:
      placement:
        constraints:
          - node.hostname == worker1
EOF

# docker-compose-worker2.yml
cat <<EOF > docker-compose2.yml
version: "3.8"

services:
  data-api-worker2:
    build: .
    image: vourteen14/data-api:latest
    ports:
      - target: 5000
        published: 8099
        protocol: tcp
        mode: ingress
    deploy:
      placement:
        constraints:
          - node.hostname == worker2
EOF
````

## build and push sample app (on master)
````
sudo docker build -t vourteen14/data-api .
sudo docker push vourteen14/data-api:latest
````

## pull image on worker1 & worker2
````
sudo docker pull vourteen14/data-api:latest
````

## start swarm service 
````
sudo docker stack deploy -c docker-compose1.yml dataapi
sudo docker stack deploy -c docker-compose2.yml dataapi

````

## setup nginx (on master)
````
sudo apt install nginx -y

# /etc/nginx/sites-available/data-api1
sudo tee /etc/nginx/sites-available/data-api1 > /dev/null <<EOF
server {
    listen 80;
    server_name data-api1.local;

    location / {
        proxy_pass http://127.0.0.1:8088;
        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header Connection "";
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;

        client_max_body_size 100m;
        proxy_read_timeout 120s;
        proxy_connect_timeout 30s;
    }
}
EOF

# /etc/nginx/sites-available/data-api2
sudo tee /etc/nginx/sites-available/data-api2 > /dev/null <<EOF
server {
    listen 80;
    server_name data-api2.local;

    location / {
        proxy_pass http://127.0.0.1:8099;
        proxy_http_version 1.1;
        proxy_set_header Host \$host;
        proxy_set_header Connection "";
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;

        client_max_body_size 100m;
        proxy_read_timeout 120s;
        proxy_connect_timeout 30s;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/data-api1 /etc/nginx/sites-enable/data-api1
sudo ln -s /etc/nginx/sites-available/data-api1 /etc/nginx/sites-enable/data-api1

sudo systemctl restart nginx
````

## add to hostname (on master or outher server that can access master swarm)
echo "10.30.11.11 data-api1.local" | sudo tee -a /etc/hosts > /dev/null
echo "10.30.11.11 data-api2.local" | sudo tee -a /etc/hosts > /dev/null

## Start reproduce MTU mismatch
### do this on another node (out of swarm cluster)
while(true); do curl -s http://data-api1.local/generate?size=10240; done

## and when hit the endpoint, do the service update
### deploy new service
sudo docker service create --name sleeper --replicas 3 --network alphin alpine sleep 1000

### make a chane such as image 
sudo docker service update --image nginx:latest sleeper

## check docker logs on master, must be there has an error like below
````
Jul 22 07:20:35 swarm-fr1 dockerd[750]: time="2025-07-22T07:20:35.691926409Z" level=info msg="initialized VXLAN UDP port to 4789 " module=node node.id=kdrq9905y4jq1b8cxhf1me1uu
Jul 22 07:20:38 swarm-fr1 dockerd[750]: time="2025-07-22T07:20:38.206005759Z" level=warning msg="reference for unknown type: " digest="sha256:4bcff63911fcb4448bd4fdacec207030997caf25e9bea4045fa6c8c44de311d1" remote="docker.io/library/alpine:latest@sha256:4bcff63911fcb4448bd4fdacec207030997caf25e9bea4045fa6c8c44de311d1"
Jul 22 07:24:27 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:27.334092285Z" level=warning msg="error deleting neighbor entry" error="no such file or directory" ifc=vx-001003-n6xad ip=10.30.11.13 mac="02:42:0a:00:03:04"
Jul 22 07:24:27 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:27.334914841Z" level=warning msg="Peer delete operation failed" error="could not delete fdb entry for nid:n6xad515dzrl5qw983m8wo0ki eid:91cae388080e9acb4237d2c94fd1a4770ce9845e8c207664dca056f7d3f5543b into the sandbox:neighbor entry not found for IP 10.30.11.13, mac 02:42:0a:00:03:04, link vx-001003-n6xad"
Jul 22 07:24:27 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:27.335178512Z" level=warning msg="rmServiceBinding 91cae388080e9acb4237d2c94fd1a4770ce9845e8c207664dca056f7d3f5543b possible transient state ok:false entries:0 set:false "
Jul 22 07:24:28 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:28.134534285Z" level=warning msg="error deleting neighbor entry" error="no such file or directory" ifc=vx-001003-n6xad ip=10.30.11.13 mac="02:42:0a:00:03:06"
Jul 22 07:24:28 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:28.135129132Z" level=warning msg="Peer delete operation failed" error="could not delete fdb entry for nid:n6xad515dzrl5qw983m8wo0ki eid:3006cdb80745041ad963a96e1818a8d311f81d0f5f04172dd5b8d1521b808776 into the sandbox:neighbor entry not found for IP 10.30.11.13, mac 02:42:0a:00:03:06, link vx-001003-n6xad"
Jul 22 07:24:29 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:29.933921346Z" level=warning msg="Neighbor entry already present" ifc=vx-001003-n6xad ip=10.0.3.6 mac="02:42:0a:00:03:06" neigh="10.0.3.6 02:42:0a:00:03:06"
Jul 22 07:24:29 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:29.934633377Z" level=warning msg="Peer add operation failed" error="could not add neighbor entry for nid:n6xad515dzrl5qw983m8wo0ki eid:518aa4385033c24dd7b732b77f72d7eb537b37329341b617c7b057247ff33013 into the sandbox:neighbor entry already exists for IP 10.0.3.6, mac 02:42:0a:00:03:06, link vx-001003-n6xad"
Jul 22 07:24:47 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:47.337353889Z" level=info msg="NetworkDB stats swarm-fr1(1d92db859747) - netID:n6xad515dzrl5qw983m8wo0ki leaving:false netPeers:3 entries:10 Queue qLen:0 netMsg/s:0"
Jul 22 07:24:47 swarm-fr1 dockerd[750]: time="2025-07-22T07:24:47.338172586Z" level=info msg="NetworkDB stats swarm-fr1(1d92db859747) - netID:a8qujiohdm624u6ihi029ljcr leaving:false netPeers:3 entries:7 Queue qLen:0 netMsg/s:0"
Jul 22 07:25:39 swarm-fr1 dockerd[750]: time="2025-07-22T07:25:39.853037523Z" level=warning msg="reference for unknown type: " digest="sha256:84ec966e61a8c7846f509da7eb081c55c1d56817448728924a87ab32f12a72fb" remote="docker.io/library/nginx:latest@sha256:84ec966e61a8c7846f509da7eb081c55c1d56817448728924a87ab32f12a72fb"
Jul 22 07:25:52 swarm-fr1 dockerd[750]: time="2025-07-22T07:25:52.154032850Z" level=info msg="Container failed to exit within 10s of signal 15 - using the force" container=37db2b706a7db19d866f70b9d150e396a2fd1629d530bd44d678332b646dfc74
Jul 22 07:25:52 swarm-fr1 dockerd[750]: time="2025-07-22T07:25:52.272189793Z" level=info msg="ignoring event" container=37db2b706a7db19d866f70b9d150e396a2fd1629d530bd44d678332b646dfc74 module=libcontainerd namespace=moby topic=/tasks/delete type="*events.TaskDelete"
Jul 22 07:25:52 swarm-fr1 dockerd[750]: time="2025-07-22T07:25:52.329297737Z" level=warning msg="rmServiceBinding 99db2fdce0bfb770e9f040026ed5d981a100e1beea7638ad18e2331e2432d496 possible transient state ok:false entries:0 set:false "
Jul 22 07:25:53 swarm-fr1 dockerd[750]: time="2025-07-22T07:25:53.602827599Z" level=warning msg="Error (Unable to complete atomic operation, key modified) deleting object [endpoint n6xad515dzrl5qw983m8wo0ki 6e6082602cb282a54046759f895af73ae984dd146e6b80ca1fed93b5cc35c3c6], retrying...."
Jul 22 07:25:53 swarm-fr1 dockerd[750]: time="2025-07-22T07:25:53.652985521Z" level=warning msg="Error (Unable to complete atomic operation, key modified) deleting object [endpoint_count n6xad515dzrl5qw983m8wo0ki], retrying...."
Jul 22 07:25:53 swarm-fr1 dockerd[750]: time="2025-07-22T07:25:53.663515669Z" level=error msg="failed removing service binding" R="{sleeper.3.sre5rpbmvgdwh96xelnzx2gfp sleeper 5rzk9v5u6im3ipf3obrtmc0fg 10.0.3.2 10.0.3.5 [] [] [11681fd9b6f8] false}" T=networkdb.DeleteEvent eid=bcb3283a505fa6842b0cd4a486d5887d02b77e6e2bc2fb65415e208ffa8e7abd error="network n6xad515dzrl5qw983m8wo0ki not found" nid=n6xad515dzrl5qw983m8wo0ki
Jul 22 07:25:53 swarm-fr1 dockerd[750]: time="2025-07-22T07:25:53.663955805Z" level=error msg="failed removing service binding" R="{sleeper.2.u1658nouvnj1dm7bzbqmw5tbf sleeper 5rzk9v5u6im3ipf3obrtmc0fg 10.0.3.2 10.0.3.9 [] [] [41f7485c25fd] false}" T=networkdb.DeleteEvent eid=e6c9ecdebf3e6be75eba74925165a7d7fa0dfa61ec6eedaf42b8d53fe28949d5 error="network n6xad515dzrl5qw983m8wo0ki not found" nid=n6xad515dzrl5qw983m8wo0ki
Jul 22 07:27:13 swarm-fr1 dockerd[750]: time="2025-07-22T07:27:13.337057145Z" level=warning msg="error deleting neighbor entry" error="no such file or directory" ifc=vx-001003-n6xad ip=10.30.11.12 mac="02:42:0a:00:03:05"
Jul 22 07:27:13 swarm-fr1 dockerd[750]: time="2025-07-22T07:27:13.338123729Z" level=warning msg="Peer delete operation failed" error="could not delete fdb entry for nid:n6xad515dzrl5qw983m8wo0ki eid:bcb3283a505fa6842b0cd4a486d5887d02b77e6e2bc2fb65415e208ffa8e7abd into the sandbox:neighbor entry not found for IP 10.30.11.12, mac 02:42:0a:00:03:05, link vx-001003-n6xad"
Jul 22 07:27:13 swarm-fr1 dockerd[750]: time="2025-07-22T07:27:13.338579518Z" level=warning msg="rmServiceBinding bcb3283a505fa6842b0cd4a486d5887d02b77e6e2bc2fb65415e208ffa8e7abd possible transient state ok:false entries:0 set:false "
Jul 22 07:27:14 swarm-fr1 dockerd[750]: time="2025-07-22T07:27:14.336074791Z" level=warning msg="error deleting neighbor entry" error="no such file or directory" ifc=vx-001003-n6xad ip=10.30.11.12 mac="02:42:0a:00:03:07"
Jul 22 07:27:14 swarm-fr1 dockerd[750]: time="2025-07-22T07:27:14.336150497Z" level=warning msg="Peer delete operation failed" error="could not delete fdb entry for nid:n6xad515dzrl5qw983m8wo0ki eid:a62f4994f3f1713411888f43a6ab2f5d7b5ca1d30ada2ef14983f223fff264f9 into the sandbox:neighbor entry not found for IP 10.30.11.12, mac 02:42:0a:00:03:07, link vx-001003-n6xad"
Jul 22 07:27:16 swarm-fr1 dockerd[750]: time="2025-07-22T07:27:16.136874248Z" level=warning msg="Neighbor entry already present" ifc=vx-001003-n6xad ip=10.0.3.7 mac="02:42:0a:00:03:07" neigh="10.0.3.7 02:42:0a:00:03:07"
Jul 22 07:27:16 swarm-fr1 dockerd[750]: time="2025-07-22T07:27:16.137458192Z" level=warning msg="Peer add operation failed" error="could not add neighbor entry for nid:n6xad515dzrl5qw983m8wo0ki eid:e4b6c245bc280cb509a114022039f80fbd5911868506020eb18f00a88553c50e into the sandbox:neighbor entry already exists for IP 10.0.3.7, mac 02:42:0a:00:03:07, link vx-001003-n6xad"
Jul 22 07:29:47 swarm-fr1 dockerd[750]: time="2025-07-22T07:29:47.536956947Z" level=info msg="NetworkDB stats swarm-fr1(1d92db859747) - netID:a8qujiohdm624u6ihi029ljcr leaving:false netPeers:3 entries:7 Queue qLen:0 netMsg/s:0"
Jul 22 07:29:47 swarm-fr1 dockerd[750]: time="2025-07-22T07:29:47.537649896Z" level=info msg="NetworkDB stats swarm-fr1(1d92db859747) - netID:n6xad515dzrl5qw983m8wo0ki leaving:false netPeers:3 entries:18 Queue qLen:0 netMsg/s:0"
````

Log:
`Peer add operation failed [...] neighbor entry already exists for IP [...]`

indicate swarm try to add new neighbor entry but i seem there as missmatch information between master and worker

## Solution
### Change overlay MTU same as core NIC
### Another solution is under research

## Cleanup on all nodes
### un-drop icmp
sudo iptables -D INPUT -p icmp -j DROP
sudo iptables -D OUTPUT -p icmp -j DROP

sudo apt purge docker-ce containerd.io docker-ce-cli

