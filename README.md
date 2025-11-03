# Secure Nginx Hardening with Ansible and Podman

This project demonstrates a minimal, reproducible approach to hardening Nginx containers using Ansible with the Podman connection plugin. The focus is on automation, security best practices, and simplicity.

## Prerequisites

Ensure the following packages are installed on your Fedora (or compatible Linux) system:

```bash
sudo dnf install -y podman ansible
```

> Note: This guide assumes a standard Fedora Workstation or Server environment with Podman and Ansible available in the repositories.

## Step 1: Create Project Files

Create the following three files in your working directory.

### `Dockerfile`
```Dockerfile
FROM nginx:alpine
RUN apk add --no-cache python3
```

### `inventory.ini`
```ini
[webservers]
test1 ansible_connection=podman ansible_python_interpreter=/usr/bin/python3
test2 ansible_connection=podman ansible_python_interpreter=/usr/bin/python3
```

### `nginx.yml`
```yaml
---
- name: Harden Nginx containers
  hosts: webservers
  tasks:
    - name: Remove default welcome page
      file:
        path: /usr/share/nginx/html/index.html
        state: absent

    - name: Ensure nginx.conf does not expose version
      lineinfile:
        path: /etc/nginx/nginx.conf
        regexp: '^\s*#?\s*server_tokens'
        line: '    server_tokens off;'
        insertafter: 'http {'
      notify: Reload nginx

  handlers:
    - name: Reload nginx
      command: nginx -s reload
```

## Step 2: Build the Custom Image

Build the hardened Nginx image with Python support (required for Ansible):

```bash
podman build -t nginx-hardening .
```

## Step 3: Create a Dedicated Network (Optional but Recommended)

```bash
podman network create localnet
```

> This isolates your containers and ensures consistent addressing.

## Step 4: Launch Containers

```bash
podman run -d --name test1 -p 8080:80 --network localnet nginx-hardening
podman run -d --name test2 -p 8081:80 --network localnet nginx-hardening
```

## Step 5: Apply Security Hardening

Run the Ansible playbook to enforce security settings:

```bash
ansible-playbook -i inventory.ini nginx.yml
```

The playbook will:
- Remove the default `index.html` file to prevent information leakage.
- Disable server version exposure by setting `server_tokens off;` in `nginx.conf`.
- Reload Nginx to apply configuration changes without restart.

## Verification

To confirm the changes:

1. **Check file removal**:
   ```bash
   podman exec test1 ls /usr/share/nginx/html/
   ```
   The `index.html` file should no longer appear.

2. **Verify configuration**:
   ```bash
   podman exec test1 grep "server_tokens" /etc/nginx/nginx.conf
   ```
   Output should be: `    server_tokens off;`

3. **Test HTTP headers**:
   ```bash
   curl -I http://localhost:8080
   ```
   The `Server` header should show only `nginx` without version number.

## Cleanup (Optional)

To remove all resources:

```bash
podman stop test1 test2
podman rm test1 test2
podman rmi nginx-hardening
podman network rm localnet
```

## Conclusion

This setup provides a foundational example of infrastructure-as-code for container hardening. By combining Podman’s rootless container capabilities with Ansible’s idempotent automation, security configurations become versionable, testable, and consistently enforceable—key principles in modern DevSecOps practices.
