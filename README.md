# Mastodon Role

This Ansible role deploys Mastodon social media server with support for both Nginx and Apache webservers.

## Features

- PostgreSQL database setup with proper locale configuration
- Node.js and Yarn package management
- Ruby gem installation with bundler
- Asset compilation with Vite
- Systemd service management
- Choice of Nginx or Apache webserver
- SSL/TLS support (Let's Encrypt ready)
- Active Record encryption key management

## Requirements

- Ansible 2.9+
- Debian/Ubuntu target systems
- PostgreSQL role (geerlingguy.postgresql)
- Node.js role (geerlingguy.nodejs) - conditionally included

## Variables

### Core Configuration

```yaml
mastodon_domain: example.com          # Required: Your Mastodon domain
mastodon_db_name: mastodon_production # Database name
mastodon_db_password: secure_password # Database password
```

### Webserver Configuration

```yaml
mastodon_webserver: nginx             # Options: "nginx" or "apache"
mastodon_enable_ssl: false            # Enable SSL configuration
mastodon_ssl_cert_path: /path/to/cert # SSL certificate path
mastodon_ssl_key_path: /path/to/key   # SSL private key path
```

### SMTP Configuration

```yaml
mastodon_smtp_server: smtp.example.com
mastodon_smtp_port: 587
mastodon_smtp_login: user@example.com
mastodon_smtp_password: smtp_password
mastodon_smtp_from_address: 'Mastodon <noreply@example.com>'
mastodon_smtp_auth_method: plain
mastodon_smtp_openssl_verify_mode: none
mastodon_smtp_enable_starttls: auto
```

### Admin User Configuration

The role can automatically create an admin user during deployment:

```yaml
mastodon_admin_email: "admin@example.com"     # Required: Admin email address
mastodon_admin_username: "admin"              # Optional: Admin username (default: "admin")
mastodon_admin_password: "secure_password"    # Optional: Admin password (auto-generated if empty)
mastodon_admin_confirmed: true                # Optional: Auto-confirm email (default: true)
mastodon_admin_approved: true                 # Optional: Auto-approve account (default: true)
```

When `mastodon_admin_email` is set:
- An admin user will be created automatically
- If no password is provided, a secure 32-character password is generated
- The user gets full admin privileges (all permissions)
- Credentials are displayed in Ansible output and saved to `/opt/mastodon/.admin_credentials`
- The admin can access the dashboard at `https://{{ mastodon_domain }}/admin`

**Security Note**: The credentials file contains sensitive information. Review and delete it after noting the credentials.

## Webserver Support

### Nginx (Default)

The role uses Nginx by default with optimized configuration for Mastodon:

- Reverse proxy to Mastodon application (port 3000)
- WebSocket support for streaming API (port 4000)
- Static asset serving with proper caching headers
- Gzip compression
- Security headers

### Apache

When using Apache (`mastodon_webserver: apache`), the role provides:

- Virtual host configuration with mod_rewrite
- Proxy support for application and streaming API
- WebSocket support via mod_proxy_wstunnel
- Static asset handling with caching
- Security headers and SSL support

Required Apache modules (automatically enabled):
- rewrite
- proxy
- proxy_http
- proxy_wstunnel
- headers
- deflate
- ssl

## Usage

### Basic Nginx Setup

```yaml
- hosts: mastodon_servers
  roles:
    - role: services/mastodon
      vars:
        mastodon_domain: social.example.com
        mastodon_webserver: nginx
```

### Complete Setup with Admin User

```yaml
- hosts: mastodon_servers
  roles:
    - role: services/mastodon
      vars:
        mastodon_domain: social.example.com
        mastodon_webserver: nginx
        mastodon_admin_email: "admin@social.example.com"
        mastodon_admin_username: "admin"
        # mastodon_admin_password: ""  # Leave empty for auto-generation
```

### Apache Setup for Existing Apache Server

```yaml
- hosts: mastodon_servers
  roles:
    - role: services/mastodon
      vars:
        mastodon_domain: social.example.com
        mastodon_webserver: apache
        mastodon_enable_ssl: true
```

### SSL Configuration

When `mastodon_enable_ssl: true`, the role expects SSL certificates to be available:

For Nginx:
- Certificate: `/etc/letsencrypt/live/{{ mastodon_domain }}/fullchain.pem`
- Private key: `/etc/letsencrypt/live/{{ mastodon_domain }}/privkey.pem`

For Apache:
- Uses the same certificate paths
- Automatically redirects HTTP to HTTPS

## Integration with Existing Apache

When Apache is already serving on ports 80/443:

1. Set `mastodon_webserver: apache`
2. The role will create a virtual host configuration
3. Enable the site using `a2ensite`
4. Required Apache modules are automatically enabled
5. The existing Apache service continues running

## Directory Structure

```
/opt/mastodon/               # Mastodon installation
├── app/                     # Application code
├── public/                  # Static assets
├── .env.production          # Environment configuration
└── vendor/                  # Ruby gems

/etc/systemd/system/         # Systemd service files
├── mastodon-web.service
├── mastodon-streaming.service
└── mastodon-sidekiq.service

# Webserver configuration
/etc/nginx/sites-available/mastodon    # Nginx
/etc/apache2/sites-available/{{ mastodon_domain }}.conf  # Apache
```

## Services

The role creates and manages these systemd services:

- `mastodon-web`: Main web application (port 3000)
- `mastodon-streaming`: Streaming API server (port 4000)  
- `mastodon-sidekiq`: Background job processor

## Troubleshooting

### Apache Module Issues

If Apache modules fail to load:

```bash
# Check enabled modules
apache2ctl -M

# Manually enable required modules
a2enmod rewrite proxy proxy_http proxy_wstunnel headers deflate ssl
systemctl reload apache2
```

### SSL Certificate Issues

For Let's Encrypt certificates:

```bash
# Generate certificate
certbot --apache -d {{ mastodon_domain }}

# Or for nginx
certbot --nginx -d {{ mastodon_domain }}
```

### Service Status

Check Mastodon services:

```bash
systemctl status mastodon-web
systemctl status mastodon-streaming  
systemctl status mastodon-sidekiq
```

### Logs

View application logs:

```bash
journalctl -u mastodon-web -f
journalctl -u mastodon-streaming -f
journalctl -u mastodon-sidekiq -f
```

## License

This role is part of the VitexSoftware Ansible infrastructure project.