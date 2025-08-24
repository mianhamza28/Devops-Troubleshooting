# Complete Guide: Managing Docker Disk Space on Linux

*"What should I do if I run out of disk space on Linux while using Docker?"*

This is one of the most common Docker issues. Docker can consume massive amounts of disk space through images, containers, volumes, logs, and build cache. This comprehensive guide will help you diagnose, clean up, and prevent disk space issues.

---

## 🔍 **Phase 1: Diagnose the Problem**

### 1.1 Check Overall System Disk Usage
```bash
# Check overall disk usage
df -h

# Check Docker's root directory specifically
sudo du -sh /var/lib/docker/
```

### 1.2 Analyze Docker Disk Usage
```bash
# Get Docker's disk usage summary
docker system df

# Get detailed breakdown
docker system df -v

# Check Docker daemon info (includes storage driver info)
docker info | grep -A 20 "Storage Driver"
```

### 1.3 Identify the Biggest Space Consumers
```bash
# Find largest Docker directories
sudo du -sh /var/lib/docker/*/ | sort -rh

# Check container logs size (often overlooked!)
sudo find /var/lib/docker/containers/ -name "*.log" -exec ls -lh {} \; | awk '{print $5, $9}' | sort -rh | head -20

# Check individual image sizes
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | sort -k 3 -h
```

---

## 🧹 **Phase 2: Safe Cleanup (Start Here)**

### 2.1 Standard Docker Cleanup
```bash
# Remove stopped containers, unused networks, and dangling images
docker system prune

# More aggressive: remove all unused images too
docker system prune -a

# Most aggressive: include unused volumes (⚠️ CAUTION!)
docker system prune -a --volumes
```

### 2.2 Targeted Cleanup Commands
```bash
# Remove only dangling images (safe)
docker image prune

# Remove all unused images (more aggressive)
docker image prune -a

# Remove stopped containers
docker container prune

# Remove unused volumes (⚠️ backup important data first!)
docker volume prune

# Remove unused networks
docker network prune

# Clean build cache
docker builder prune
docker builder prune -a  # More aggressive
```

---

## 🎯 **Phase 3: Advanced Cleanup Techniques**

### 3.1 Manual Selective Removal
```bash
# List and remove specific images
docker images
docker rmi $(docker images -q -f "dangling=true")  # Remove dangling images
docker rmi $(docker images -q --filter "before=IMAGE_NAME")  # Remove images older than specified

# List and remove specific containers
docker ps -a
docker rm $(docker ps -aq -f "status=exited")  # Remove all exited containers

# List and remove specific volumes
docker volume ls
docker volume ls -q -f "dangling=true" | xargs docker volume rm  # Remove dangling volumes
```

### 3.2 Clean Container Logs (Often Overlooked!)
```bash
# Find largest log files
sudo find /var/lib/docker/containers/ -name "*.log" -exec ls -lsh {} \; | sort -k 1 -hr | head -20

# Truncate all container logs
sudo truncate -s 0 /var/lib/docker/containers/*/*-json.log

# Or clean logs for specific container
docker logs CONTAINER_NAME > /dev/null 2>&1
sudo truncate -s 0 $(docker inspect CONTAINER_NAME | grep LogPath | cut -d'"' -f4)
```

### 3.3 Remove Everything and Start Fresh (Nuclear Option)
```bash
# ⚠️ WARNING: This removes EVERYTHING Docker-related
docker system prune -a --volumes --force
docker builder prune -a --force

# If that's not enough, stop Docker and remove everything manually
sudo systemctl stop docker
sudo rm -rf /var/lib/docker/
sudo systemctl start docker
```

---

## ⚙️ **Phase 4: Advanced Techniques & Troubleshooting**

### 4.1 Handle Stubborn Images/Containers
```bash
# Force remove images
docker rmi -f IMAGE_ID

# Remove containers using image
docker ps -a | grep IMAGE_NAME | awk '{print $1}' | xargs docker rm -f

# Remove all containers (running and stopped)
docker rm -f $(docker ps -aq)

# Remove all images
docker rmi -f $(docker images -q)
```

### 4.2 Check for Hidden Space Usage
```bash
# Check Docker's storage driver overlay2 usage
sudo du -sh /var/lib/docker/overlay2/* | sort -rh | head -20

# Find largest directories in Docker root
sudo du -h --max-depth=2 /var/lib/docker/ | sort -rh | head -20

# Check for orphaned data
docker system df
docker system events --filter type=container --since 1h  # Check recent activity
```

### 4.3 Registry and Cache Cleanup
```bash
# If using local registry, clean it
docker exec REGISTRY_CONTAINER registry garbage-collect /etc/docker/registry/config.yml

# Clean Docker buildx cache
docker buildx prune -a
```

---

## 🛡️ **Phase 5: Prevention & Best Practices**

### 5.1 Configure Log Rotation
```bash
# Create/edit Docker daemon config
sudo nano /etc/docker/daemon.json
```

Add log rotation settings:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
```

```bash
# Restart Docker to apply changes
sudo systemctl restart docker
```

### 5.2 Set Up Automatic Cleanup
```bash
# Create a cleanup script
sudo nano /usr/local/bin/docker-cleanup.sh
```

Add this content:
```bash
#!/bin/bash
echo "Starting Docker cleanup..."
docker system prune -f
docker image prune -a -f
docker builder prune -f
echo "Docker cleanup completed!"
```

```bash
# Make it executable
sudo chmod +x /usr/local/bin/docker-cleanup.sh

# Add to crontab for weekly cleanup
echo "0 2 * * 0 /usr/local/bin/docker-cleanup.sh" | sudo crontab -
```

### 5.3 Best Practices for Docker Usage
```bash
# Use .dockerignore to reduce build context
echo "node_modules
*.log
.git" > .dockerignore

# Use multi-stage builds to reduce image size
# Use --no-cache for one-time builds
docker build --no-cache -t myapp .

# Remove intermediate containers during build
docker build --rm -t myapp .

# Use specific image tags instead of 'latest'
docker pull ubuntu:20.04  # instead of ubuntu:latest
```

### 5.4 Monitor Disk Usage
```bash
# Set up monitoring script
sudo nano /usr/local/bin/docker-monitor.sh
```

```bash
#!/bin/bash
THRESHOLD=80
USAGE=$(df /var/lib/docker | tail -1 | awk '{print $5}' | sed 's/%//')

if [ $USAGE -gt $THRESHOLD ]; then
    echo "Warning: Docker disk usage is ${USAGE}%"
    docker system df
    # Could trigger cleanup here
fi
```

---

## ⚡ **Quick Emergency Commands**

When you're completely out of space and need immediate relief:

```bash
# Emergency cleanup sequence (run in order)
docker system prune -f                          # Quick safe cleanup
docker image prune -a -f                        # Remove unused images
docker builder prune -a -f                      # Clear build cache
sudo truncate -s 0 /var/lib/docker/containers/*/*-json.log  # Clear logs

# If still not enough space:
docker system prune -a --volumes -f             # Nuclear option
```

---

## 📊 **Useful Monitoring Commands**

```bash
# Create an alias for quick Docker disk check
echo 'alias docker-space="docker system df && echo && df -h /var/lib/docker"' >> ~/.bashrc

# Monitor Docker processes and their resource usage
docker stats

# Watch disk usage in real-time
watch -n 5 'docker system df && echo && df -h /var/lib/docker'
```

---

## 🚨 **Common Pitfalls & Warnings**

1. **Volume Data Loss**: The `--volumes` flag in `docker system prune` will delete named volumes. Always backup important data first.

2. **Running Containers**: Some cleanup commands won't work if containers are running. Stop them first if safe to do so.

3. **Shared Layers**: Removing images might affect other images that share layers. Docker will warn you about this.

4. **Log Files**: Container logs can grow enormous and aren't cleaned by standard prune commands.

5. **Build Context**: Large build contexts (including unnecessary files) can fill up disk space quickly.

---

## 🎯 **Summary: Quick Decision Tree**

**Low disk space? Start here:**

1. **Immediate relief needed?** → `docker system prune -f`
2. **Need more space?** → `docker system prune -a -f`
3. **Still need more?** → `docker system prune -a --volumes -f` (⚠️ backup first!)
4. **Logs taking space?** → `sudo truncate -s 0 /var/lib/docker/containers/*/*-json.log`
5. **Still not enough?** → Consider moving Docker root or adding more disk space

**For long-term health:**
- Set up log rotation
- Schedule regular cleanups
- Monitor disk usage
- Use proper Docker practices

This comprehensive approach should resolve most Docker disk space issues and prevent them from recurring.
