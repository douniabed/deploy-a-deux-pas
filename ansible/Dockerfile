# Use Jenkins agent with JDK 21 as base image
FROM jenkins/agent:latest-jdk21

# Switch to root to install system packages
USER root

# Install minimal system dependencies for Ansible and SSH connectivity
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    python3-minimal \
    python3-pip \
    python3-venv \
    openssh-client \
    sshpass \
    rsync \
    && rm -rf /var/lib/apt/lists/*

# Create necessary directories with proper ownership
RUN mkdir -p /home/jenkins/.local && \
    mkdir -p /home/jenkins/.ansible && \
    mkdir -p /home/jenkins/.ssh && \
    chown -R jenkins:jenkins /home/jenkins/

# Switch to jenkins user for all further installations
USER jenkins

# Set up minimal Python virtual environment
RUN python3 -m venv /home/jenkins/ansible-venv

# Install minimal Ansible without extra dependencies
RUN /home/jenkins/ansible-venv/bin/pip install --no-cache-dir --upgrade pip && \
    /home/jenkins/ansible-venv/bin/pip install --no-cache-dir \
    ansible \
    paramiko \
    jinja2

# Set PATH to include ansible binaries
ENV PATH="/home/jenkins/ansible-venv/bin:${PATH}"

# Install only required Ansible collections for app deployment
RUN /home/jenkins/ansible-venv/bin/ansible-galaxy collection install \
    ansible.posix \
    community.general

# Create minimal Ansible configuration
RUN echo "[defaults]" > /home/jenkins/.ansible.cfg && \
    echo "host_key_checking = False" >> /home/jenkins/.ansible.cfg && \
    echo "interpreter_python = auto_silent" >> /home/jenkins/.ansible.cfg && \
    echo "collections_paths = /home/jenkins/.ansible/collections" >> /home/jenkins/.ansible.cfg && \
    echo "retry_files_enabled = False" >> /home/jenkins/.ansible.cfg && \
    echo "gathering = smart" >> /home/jenkins/.ansible.cfg && \
    echo "timeout = 30" >> /home/jenkins/.ansible.cfg && \
    echo "" >> /home/jenkins/.ansible.cfg && \
    echo "[ssh_connection]" >> /home/jenkins/.ansible.cfg && \
    echo "pipelining = True" >> /home/jenkins/.ansible.cfg && \
    echo "ssh_args = -o ControlMaster=auto -o ControlPersist=60s" >> /home/jenkins/.ansible.cfg

# Set environment variables
ENV ANSIBLE_CONFIG=/home/jenkins/.ansible.cfg
ENV ANSIBLE_HOST_KEY_CHECKING=False
ENV PYTHONUNBUFFERED=1

# Set working directory
WORKDIR /home/jenkins

# Add labels
LABEL description="Minimal Jenkins agent with Ansible for Spring Boot, Node.js, and MariaDB deployments" \
      ansible.version="core" \
      purpose="vm-app-deployment"

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD /home/jenkins/ansible-venv/bin/ansible --version || exit 1