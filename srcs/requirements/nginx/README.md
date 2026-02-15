First, we need to tell NGINX how to behave. We will create a specific configuration that listens on port 443 (HTTPS) and strictly limits the TLS versions.

in `default.conf`
```bash 
server {
    # Listen on port 443 for SSL/HTTPS traffic
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name localhost;

    # SSL Certificate locations (we will generate these in the Dockerfile)
    ssl_certificate /etc/nginx/ssl/self-signed.crt;
    ssl_certificate_key /etc/nginx/ssl/self-signed.key;

    # --- SECURITY SETTINGS ---
    # Strictly enforce TLSv1.2 and TLSv1.3. 
    # This disables older, insecure protocols like SSLv3, TLS 1.0, and TLS 1.1.
    ssl_protocols TLSv1.2 TLSv1.3;

    # Optimize SSL cipher suites (Recommended high-security ciphers)
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    # SSL Session settings for performance
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
        # Return a simple 200 OK text for testing
        return 200 'Hello! This traffic is secured with TLS 1.2 or 1.3.';
        add_header Content-Type text/plain;
    }
}

# Redirect HTTP (port 80) to HTTPS (port 443)
server {
    listen 80;
    listen [::]:80;
    server_name localhost;
    return 301 https://$host$request_uri;
}
```

In Dockerfile 

```bash

# 1. Base Image
# We use Alpine Linux (Stable) for a tiny footprint and reduced attack surface.
FROM alpine:latest

# 3. Installation
# We update the package index and install NGINX and OpenSSL.
# --no-cache ensures we don't store the index, keeping the image small.
RUN apk add --no-cache nginx openssl

# 4. Setup Directories
# Create the directory where we will store the certificates.
RUN mkdir -p /etc/nginx/ssl

# 5. Generate Self-Signed Certificate
# We generate the cert INSIDE the image for this demo.
# In production, you would typically mount these from the host.
RUN openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/self-signed.key \
    -out /etc/nginx/ssl/self-signed.crt \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost"

# 6. Configuration
# Copy our custom configuration file from the host into the container.
COPY default.conf /etc/nginx/http.d/default.conf

# 7. Ports
# Expose HTTP and HTTPS ports.
EXPOSE 80 443

# 8. Command
# Run NGINX in the foreground so the container doesn't exit immediately.
CMD ["nginx", "-g", "daemon off;"]
```