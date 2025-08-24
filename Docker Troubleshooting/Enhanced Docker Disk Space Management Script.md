#!/bin/bash

# Docker Disk Space Management Script
# Complete automated solution for managing Docker disk space on Linux
# Author: Generated from Complete Guide
# Version: 3.0

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
CYAN='\033[0;36m'
WHITE='\033[1;37m'
NC='\033[0m' # No Color

# Global variables
SCRIPT_NAME="Docker Disk Cleanup"
LOG_FILE="/var/log/docker-cleanup.log"
BACKUP_DIR="/tmp/docker-backup-$(date +%Y%m%d-%H%M%S)"
DOCKER_ROOT=$(docker info 2>/dev/null | grep "Docker Root Dir" | cut -d' ' -f4)
[ -z "$DOCKER_ROOT" ] && DOCKER_ROOT="/var/lib/docker"

# Utility functions
log() {
    echo -e "${GREEN}[$(date '+%Y-%m-%d %H:%M:%S')]${NC} $1" | tee -a "$LOG_FILE"
}

warn() {
    echo -e "${YELLOW}[WARNING]${NC} $1" | tee -a "$LOG_FILE"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1" | tee -a "$LOG_FILE"
}

info() {
    echo -e "${BLUE}[INFO]${NC} $1" | tee -a "$LOG_FILE"
}

success() {
    echo -e "${GREEN}[SUCCESS]${NC} $1" | tee -a "$LOG_FILE"
}

# Function to check if Docker is running
check_docker() {
    if ! command -v docker &> /dev/null; then
        error "Docker is not installed!"
        exit 1
    fi
    
    if ! systemctl is-active --quiet docker; then
        error "Docker service is not running!"
        echo "Start Docker with: sudo systemctl start docker"
        exit 1
    fi
    
    if ! docker info &> /dev/null; then
        error "Cannot connect to Docker daemon. Run as root or add user to docker group."
        exit 1
    fi
}

# Function to display header
show_header() {
    clear
    echo -e "${CYAN}=====================================================${NC}"
    echo -e "${WHITE}        Docker Disk Space Management Tool${NC}"
    echo -e "${CYAN}=====================================================${NC}"
    echo ""
}

# Function to show disk usage analysis
analyze_disk_usage() {
    info "Analyzing disk usage..."
    echo ""
    
    echo -e "${BLUE}=== System Disk Usage ===${NC}"
    df -h | head -n 1
    df -h | grep -E "(/$|/var|/docker)"
    echo ""
    
    echo -e "${BLUE}=== Docker Root Directory Usage ===${NC}"
    sudo du -sh "$DOCKER_ROOT/" 2>/dev/null || echo "Permission denied - run as root"
    echo ""
    
    echo -e "${BLUE}=== Docker System Usage ===${NC}"
    docker system df
    echo ""
    
    echo -e "${BLUE}=== Detailed Docker Usage ===${NC}"
    docker system df -v
    echo ""
    
    echo -e "${BLUE}=== Container Log Sizes ===${NC}"
    if sudo find "$DOCKER_ROOT/containers/" -name "*.log" -exec ls -lsh {} \; 2>/dev/null | sort -k 1 -hr | head -10; then
        echo ""
    else
        warn "Cannot access container logs - run as root for complete analysis"
    fi
    
    echo -e "${BLUE}=== Largest Docker Directories ===${NC}"
    sudo du -sh "$DOCKER_ROOT"/*/ 2>/dev/null | sort -rh | head -10 || warn "Run as root for detailed directory analysis"
    echo ""
}

# Function for safe cleanup
safe_cleanup() {
    info "Starting safe cleanup..."
    
    echo -e "${YELLOW}Removing stopped containers, unused networks, and dangling images...${NC}"
    docker system prune -f
    
    echo -e "${YELLOW}Removing dangling images...${NC}"
    docker image prune -f
    
    echo -e "${YELLOW}Removing stopped containers...${NC}"
    docker container prune -f
    
    echo -e "${YELLOW}Removing unused networks...${NC}"
    docker network prune -f
    
    echo -e "${YELLOW}Removing build cache...${NC}"
    docker builder prune -f
    
    success "Safe cleanup completed!"
}

# Function for aggressive cleanup
aggressive_cleanup() {
    warn "Starting aggressive cleanup..."
    warn "This will remove ALL unused images, containers, networks, and build cache!"
    
    read -p "Are you sure? This cannot be undone! (yes/NO): " confirm
    if [[ $confirm != "yes" ]]; then
        info "Aggressive cleanup cancelled."
        return
    fi
    
    echo -e "${YELLOW}Removing all unused Docker objects...${NC}"
    docker system prune -a -f
    
    echo -e "${YELLOW}Removing all unused images...${NC}"
    docker image prune -a -f
    
    echo -e "${YELLOW}Removing all build cache...${NC}"
    docker builder prune -a -f
    
    success "Aggressive cleanup completed!"
}

# Function for nuclear cleanup (including volumes)
nuclear_cleanup() {
    error "NUCLEAR OPTION: This will remove EVERYTHING unused including VOLUMES!"
    error "ALL DATA IN UNUSED VOLUMES WILL BE PERMANENTLY LOST!"
    
    read -p "Type 'I UNDERSTAND THE RISKS' to continue: " confirm
    if [[ $confirm != "I UNDERSTAND THE RISKS" ]]; then
        info "Nuclear cleanup cancelled."
        return
    fi
    
    echo -e "${RED}Creating backup directory: $BACKUP_DIR${NC}"
    mkdir -p "$BACKUP_DIR"
    
    echo -e "${YELLOW}Backing up volume list...${NC}"
    docker volume ls > "$BACKUP_DIR/volumes_before_cleanup.txt"
    
    echo -e "${RED}Removing ALL unused Docker objects including volumes...${NC}"
    docker system prune -a --volumes -f
    
    success "Nuclear cleanup completed! Backup created at: $BACKUP_DIR"
}

# Function to clean container logs
clean_container_logs() {
    info "Cleaning container logs..."
    
    # Check log sizes first
    echo -e "${BLUE}Current largest log files:${NC}"
    if sudo find "$DOCKER_ROOT/containers/" -name "*.log" -exec ls -lsh {} \; 2>/dev/null | sort -k 1 -hr | head -5; then
        echo ""
        read -p "Proceed with log cleanup? (y/N): " confirm
        if [[ $confirm =~ ^[Yy]$ ]]; then
            echo -e "${YELLOW}Truncating all container logs...${NC}"
            sudo truncate -s 0 "$DOCKER_ROOT"/containers/*/*-json.log 2>/dev/null && success "Container logs cleaned!" || warn "Could not clean logs - check permissions"
        fi
    else
        warn "Cannot access container logs - run as root"
    fi
}

# Function for manual cleanup
manual_cleanup() {
    info "Manual cleanup options:"
    echo ""
    echo "1. Remove specific images"
    echo "2. Remove specific containers"
    echo "3. Remove specific volumes"
    echo "4. Remove exited containers"
    echo "5. Remove dangling volumes"
    echo "6. Back to main menu"
    echo ""
    
    read -p "Choose option (1-6): " choice
    
    case $choice in
        1)
            docker images
            echo ""
            read -p "Enter image ID or name to remove: " image_id
            if [[ -n $image_id ]]; then
                docker rmi "$image_id" && success "Image removed!" || error "Failed to remove image"
            fi
            ;;
        2)
            docker ps -a
            echo ""
            read -p "Enter container ID or name to remove: " container_id
            if [[ -n $container_id ]]; then
                docker rm -f "$container_id" && success "Container removed!" || error "Failed to remove container"
            fi
            ;;
        3)
            docker volume ls
            echo ""
            read -p "Enter volume name to remove: " volume_name
            if [[ -n $volume_name ]]; then
                read -p "Are you sure? This will delete all data in the volume! (y/N): " confirm
                if [[ $confirm =~ ^[Yy]$ ]]; then
                    docker volume rm "$volume_name" && success "Volume removed!" || error "Failed to remove volume"
                fi
            fi
            ;;
        4)
            echo -e "${YELLOW}Removing all exited containers...${NC}"
            docker rm $(docker ps -aq -f "status=exited") 2>/dev/null && success "Exited containers removed!" || info "No exited containers found"
            ;;
        5)
            echo -e "${YELLOW}Removing dangling volumes...${NC}"
            docker volume ls -q -f "dangling=true" | xargs docker volume rm 2>/dev/null && success "Dangling volumes removed!" || info "No dangling volumes found"
            ;;
        6)
            return
            ;;
        *)
            error "Invalid option!"
            ;;
    esac
}

# Function to setup log rotation
setup_log_rotation() {
    info "Setting up Docker log rotation..."
    
    # Create daemon.json if it doesn't exist
    if [ ! -f "/etc/docker/daemon.json" ]; then
        sudo mkdir -p /etc/docker
        echo '{}' | sudo tee /etc/docker/daemon.json > /dev/null
    fi
    
    # Backup current config
    sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.backup
    
    # Create new config with log rotation using jq if available
    if command -v jq &> /dev/null; then
        sudo jq '. + {"log-driver": "json-file", "log-opts": {"max-size": "10m", "max-file": "5"}}' /etc/docker/daemon.json > /tmp/daemon.json.new
        sudo mv /tmp/daemon.json.new /etc/docker/daemon.json
    else
        warn "jq not found, using basic JSON manipulation"
        cat << 'EOF' | sudo tee /etc/docker/daemon.json.new > /dev/null
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
EOF
        sudo mv /etc/docker/daemon.json.new /etc/docker/daemon.json
    fi
    
    warn "Docker daemon configuration updated. Restart required!"
    read -p "Restart Docker now? (y/N): " restart
    if [[ $restart =~ ^[Yy]$ ]]; then
        sudo systemctl restart docker
        success "Docker restarted with log rotation enabled!"
    else
        warn "Please restart Docker manually: sudo systemctl restart docker"
    fi
}

# Function to setup automated cleanup
setup_automated_cleanup() {
    info "Setting up automated cleanup..."
    
    # Create cleanup script
    cat << 'EOF' > /tmp/docker-auto-cleanup.sh
#!/bin/bash
# Automated Docker cleanup script
LOG_FILE="/var/log/docker-auto-cleanup.log"

echo "[$(date)] Starting automated Docker cleanup..." >> "$LOG_FILE"
docker system prune -f >> "$LOG_FILE" 2>&1
docker image prune -f >> "$LOG_FILE" 2>&1
docker builder prune -f >> "$LOG_FILE" 2>&1
echo "[$(date)] Automated Docker cleanup completed!" >> "$LOG_FILE"
EOF
    
    sudo mv /tmp/docker-auto-cleanup.sh /usr/local/bin/docker-auto-cleanup.sh
    sudo chmod +x /usr/local/bin/docker-auto-cleanup.sh
    
    # Setup crontab
    echo "0 2 * * 0 /usr/local/bin/docker-auto-cleanup.sh" | sudo crontab -
    
    success "Automated cleanup setup complete! Will run every Sunday at 2 AM"
    success "Script location: /usr/local/bin/docker-auto-cleanup.sh"
    success "Logs will be written to: /var/log/docker-auto-cleanup.log"
}

# Function to show monitoring commands
show_monitoring() {
    info "Useful monitoring commands and aliases:"
    echo ""
    echo -e "${CYAN}Add these aliases to your ~/.bashrc:${NC}"
    echo 'alias docker-space="docker system df && echo && df -h /var/lib/docker"'
    echo 'alias docker-clean="docker system prune -f"'
    echo 'alias docker-logs="sudo find /var/lib/docker/containers/ -name \"*.log\" -exec ls -lsh {} \; | sort -k 1 -hr | head -10"'
    echo ""
    echo -e "${CYAN}Real-time monitoring:${NC}"
    echo 'watch -n 5 "docker system df && echo && df -h /var/lib/docker"'
    echo ""
    echo -e "${CYAN}Container resource usage:${NC}"
    echo "docker stats"
    echo ""
}

# Emergency cleanup function
emergency_cleanup() {
    error "EMERGENCY CLEANUP MODE"
    warn "This will perform the most aggressive cleanup possible!"
    
    read -p "Continue with emergency cleanup? (yes/NO): " confirm
    if [[ $confirm != "yes" ]]; then
        info "Emergency cleanup cancelled."
        return
    fi
    
    info "Step 1: Standard cleanup..."
    docker system prune -f
    
    info "Step 2: Removing unused images..."
    docker image prune -a -f
    
    info "Step 3: Clearing build cache..."
    docker builder prune -a -f
    
    info "Step 4: Cleaning container logs..."
    sudo truncate -s 0 "$DOCKER_ROOT"/containers/*/*-json.log 2>/dev/null
    
    echo ""
    info "Emergency cleanup completed. Checking results..."
    echo ""
    docker system df
}

# Function to check and backup specific volumes before removal
backup_volume() {
    local volume_name=$1
    local backup_path="$BACKUP_DIR/$volume_name.tar.gz"
    
    info "Backing up volume '$volume_name' to $backup_path"
    docker run --rm -v "$volume_name":/volume -v "$BACKUP_DIR":/backup alpine \
        tar -czf "/backup/$volume_name.tar.gz" -C /volume . 2>/dev/null
    
    if [ $? -eq 0 ]; then
        success "Volume backup completed: $backup_path"
    else
        error "Failed to backup volume: $volume_name"
    fi
}

# Function to cleanup unused volumes with backup option
cleanup_volumes() {
    info "Cleaning up unused volumes..."
    
    read -p "Backup volumes before removal? (y/N): " backup_choice
    if [[ $backup_choice =~ ^[Yy]$ ]]; then
        mkdir -p "$BACKUP_DIR"
        docker volume ls -q -f "dangling=true" | while read volume; do
            backup_volume "$volume"
        done
    fi
    
    read -p "Remove all unused volumes? (y/N): " remove_choice
    if [[ $remove_choice =~ ^[Yy]$ ]]; then
        docker volume ls -q -f "dangling=true" | xargs docker volume rm 2>/dev/null
        success "Unused volumes removed!"
    fi
}

# Main menu function
show_menu() {
    echo -e "${WHITE}Choose an option:${NC}"
    echo ""
    echo -e "${GREEN} 1)${NC} Analyze Disk Usage"
    echo -e "${GREEN} 2)${NC} Safe Cleanup (Recommended)"
    echo -e "${YELLOW} 3)${NC} Aggressive Cleanup"
    echo -e "${RED} 4)${NC} Nuclear Cleanup (Includes Volumes!)"
    echo -e "${BLUE} 5)${NC} Clean Container Logs"
    echo -e "${PURPLE} 6)${NC} Manual Cleanup Options"
    echo -e "${CYAN} 7)${NC} Setup Log Rotation"
    echo -e "${CYAN} 8)${NC} Setup Automated Cleanup"
    echo -e "${WHITE} 9)${NC} Show Monitoring Commands"
    echo -e "${RED}10)${NC} Emergency Cleanup"
    echo -e "${BLUE}11)${NC} Cleanup Unused Volumes"
    echo -e "${WHITE}12)${NC} Exit"
    echo ""
}

# Main script execution
main() {
    # Check if running as root for certain operations
    if [[ $EUID -eq 0 ]]; then
        warn "Running as root - full functionality available"
    else
        info "Running as user - some features require root access"
    fi
    
    # Create log file
    sudo touch "$LOG_FILE" 2>/dev/null || touch "$LOG_FILE"
    
    log "Docker Disk Cleanup Script started"
    
    while true; do
        show_header
        check_docker
        show_menu
        
        read -p "Enter your choice (1-12): " choice
        echo ""
        
        case $choice in
            1)
                analyze_disk_usage
                ;;
            2)
                safe_cleanup
                ;;
            3)
                aggressive_cleanup
                ;;
            4)
                nuclear_cleanup
                ;;
            5)
                clean_container_logs
                ;;
            6)
                manual_cleanup
                ;;
            7)
                setup_log_rotation
                ;;
            8)
                setup_automated_cleanup
                ;;
            9)
                show_monitoring
                ;;
            10)
                emergency_cleanup
                ;;
            11)
                cleanup_volumes
                ;;
            12)
                log "Docker Disk Cleanup Script ended"
                echo -e "${GREEN}Thank you for using Docker Disk Cleanup Script!${NC}"
                exit 0
                ;;
            *)
                error "Invalid option! Please choose 1-12."
                ;;
        esac
        
        echo ""
        read -p "Press Enter to continue..."
    done
}

# Script entry point
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
