# Apache HTTP Server Configuration for STIG Manager

This directory contains Apache HTTP Server configuration files for the STIG Manager orchestration with CAC authentication support.

## Files

- **httpd.conf** - Main Apache configuration file
- **index.html** - Landing page with CAC certificate information display (uses SSI)

## Configuration Overview

The Apache configuration provides the same functionality as the nginx configuration:

### SSL/TLS Configuration

- Listens on port 443 with TLS
- Server certificate: `/usr/local/apache2/conf/cert.pem`
- Server private key: `/usr/local/apache2/conf/privkey.pem`
- Client CA certificates: `/usr/local/apache2/conf/dod-certs.pem` (DoD PKI chain)

### Client Certificate Verification

- **Default behavior**: Client certificates are optional (`SSLVerifyClient optional`)
- **OAuth endpoint**: Client certificates are required (`SSLVerifyClient require`)
  - Endpoint: `/kc/realms/stigman/protocol/openid-connect/auth`
  - Returns 403 if client cert verification fails

### Reverse Proxy Configuration

The Apache server proxies requests to backend services:

1. **STIG Manager** (`/stigman/` → `http://stigman:54000/`)
   - Preserves host headers
   - Forwards client certificate via `ssl-client-cert` header

2. **Keycloak** (`/kc/` → `http://keycloak:8080/`)
   - Preserves host headers
   - Forwards standard proxy headers:
     - `X-Forwarded-For`
     - `X-Forwarded-Host`
     - `X-Forwarded-Server`
     - `X-Forwarded-Port`
     - `X-Forwarded-Proto`
   - Forwards client certificate via `ssl-client-cert` header

### Key Apache Modules Used

- **mod_ssl** - SSL/TLS support and client certificate verification
- **mod_proxy** - Core proxy functionality
- **mod_proxy_http** - HTTP/HTTPS proxying protocol support
- **mod_headers** - Header manipulation (forwarding client cert and proxy headers)
- **mod_include** - Server Side Includes (SSI) for index.html
- **mod_autoindex** - Directory listing

## Certificate Forwarding

The client certificate is forwarded to backend services via the `ssl-client-cert` header using the Apache SSL environment variable `SSL_CLIENT_CERT`. This is set via:

```apache
RequestHeader set ssl-client-cert "%{SSL_CLIENT_CERT}s"
```

Keycloak is configured to look for this header via:
- `KC_SPI_X509CERT_LOOKUP_PROVIDER=nginx`
- `KC_SPI_X509CERT_LOOKUP_NGINX_SSL_CLIENT_CERT=SSL-CLIENT-CERT`

Note: The provider name "nginx" refers to the header-based certificate lookup mechanism, not the specific proxy software. Apache passes the certificate using the same header name.

## Differences from Nginx Configuration

While functionally equivalent, there are some implementation differences:

1. **Error codes**: Apache returns 403 (Forbidden) for failed client cert verification instead of nginx's 496 (SSL Certificate Required)

2. **SSI syntax**: Apache uses `<!--#echo var="SSL_CLIENT_CERT" -->` vs nginx's `<!--# echo var="ssl_client_raw_cert" -->`

3. **Configuration structure**: Apache uses `<Location>` and `<Directory>` blocks vs nginx's `location` blocks

4. **Certificate verification scope**: Apache's `SSLVerifyClient require` in nested Location blocks overrides the parent VirtualHost setting

## Testing

To test the Apache configuration:

```bash
cd stigman-orchestration
docker-compose up
```

Access the application at:
- STIG Manager: https://localhost/stigman/
- Keycloak Admin: https://localhost/kc/admin
- Landing page: https://localhost/

The landing page will display your CAC certificate information if SSI is working correctly.

## Troubleshooting

View Apache logs:
```bash
docker-compose logs apache
```

Common issues:
- **Module not loaded**: Ensure all required modules are loaded in httpd.conf
- **Certificate not forwarded**: Check that `SSLOptions +ExportCertData` is set
- **SSI not working**: Verify `mod_include` is loaded and `AddOutputFilter INCLUDES .html` is configured
- **Proxy connection refused**: Ensure backend services (stigman, keycloak) are running and network connectivity is correct
