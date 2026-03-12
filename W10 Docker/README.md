# Week 10

Before starting the Docker tutorial, create an AWS EC2 instance with these settings:

1. Launch a new EC2 instance using an Ubuntu AMI.
2. Choose a medium instance type.
3. Set the storage size to 20 GB.
4. Under IAM instance profile, select `LabInstanceProfile`.
5. Create a new key pair and download it so you can connect to the VM.

After the instance is running, connect to it with SSH and update the system:

```bash
sudo apt update
sudo apt upgrade -y
```

Then install Docker on the Ubuntu VM:

```bash
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```

After running the last command, sign out and sign back in before using Docker without `sudo`.

Great tutorial: https://www.docker.com/101-tutorial/

https://medium.com/@Shamimw/docker-a-complete-tutorial-part-1-basics-architecture-use-cases-96020dccb401
