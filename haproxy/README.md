# HAProxy Configuration for STIG Manager

This directory contains HAProxy configuration files for the STIG Manager orchestration with CAC authentication support.

## Files

- **haproxy.cfg** - Main HAProxy configuration file
- **combined.pem** - Combined certificate and key file (generated from localhost.crt + localhost.key)
- **index.html** - Landing page (optional)

## Configuration Overview

The HAProxy configuration provides the same functionality as the nginx and Apache configurations:

### SSL/TLS Configuration

- Listens on port 443 with TLS
- Combined certificate file: `/usr/local/etc/haproxy/certs/combined.pem` (cert + key)
- Client CA certificates: `/usr/local/etc/haproxy/certs/dod-certs.pem` (DoD PKI chain)

### Client Certificate Verification

- **Default behavior**: Client certificates are optional (`verify optional`)
- **OAuth endpoint**: Client certificates are required
  - Endpoint: `/kc/realms/stigman/protocol/openid-connect/auth`
  - Returns 403 if client cert is not present or verification fails

### Reverse Proxy Configuration

The HAProxy server proxies requests to backend services using path-based routing:

1. **STIG Manager** (`/stigman/` → `http://stigman:54000/`)
   - Strips `/stigman` prefix before forwarding
   - Forwards client certificate via `ssl-client-cert` header

2. **Keycloak** (`/kc/` → `http://keycloak:8080/`)
   - Strips `/kc` prefix before forwarding
   - Preserves host headers
   - Forwards standard proxy headers:
     - `X-Forwarded-For`
     - `X-Forwarded-Host`
     - `X-Forwarded-Server`
     - `X-Forwarded-Port`
     - `X-Forwarded-Proto`
   - Forwards client certificate via `ssl-client-cert` header

### Key HAProxy Features Used

- **Frontend/Backend separation** - Clean separation of concerns
- **ACLs** - Access Control Lists for routing and security
- **SSL termination** - `bind *:443 ssl` with certificate and CA verification
- **Sample fetch functions** - `ssl_c_der`, `ssl_c_used`, `ssl_fc_has_crt`, `ssl_c_verify`
- **Converters** - `base64` for certificate encoding
- **http-request rules** - Header manipulation and request denial

## Certificate Forwarding

The client certificate is forwarded to backend services via the `ssl-client-cert` header in base64-encoded DER format:

```haproxy
http-request set-header ssl-client-cert %[ssl_c_der,base64] if { ssl_c_used }
```

This format matches what Keycloak expects with the `apache` provider:
- `KC_SPI_X509CERT_LOOKUP_PROVIDER=apache`
- `KC_SPI_X509CERT_LOOKUP_APACHE_SSL_CLIENT_CERT=ssl-client-cert`

## Differences from Other Proxies

### vs. Nginx
- **Certificate format**: HAProxy uses DER format (binary), nginx uses PEM with tab encoding
- **Configuration style**: HAProxy uses frontend/backend separation vs nginx's location blocks
- **ACLs**: HAProxy has explicit ACL definitions vs nginx's inline conditionals
- **Performance**: HAProxy is generally more performant for high-traffic scenarios

### vs. Apache
- **Certificate file**: HAProxy requires cert+key in one file vs Apache's separate files
- **Module system**: HAProxy is monolithic vs Apache's modular architecture
- **Configuration complexity**: HAProxy syntax is simpler for load balancing scenarios

## Running the Orchestration

### Using the HAProxy configuration:

```bash
cd stigman-orchestration
docker-compose up
```

Note: The main `docker-compose.yml` now uses HAProxy by default.

Wait for the startup message from STIG Manager API:
```json
{"date":"...","level":3,"component":"index","type":"started","data":{"durationS":...,"port":54000,...}}
```

### Accessing the application

- STIG Manager: https://localhost/stigman/
- Keycloak Admin: https://localhost/kc/admin (admin/Pa55w0rd)

## Troubleshooting

View HAProxy logs:
```bash
docker-compose logs haproxy
```

Common issues:

- **Certificate not found**: Ensure `combined.pem` exists in the haproxy directory
- **Connection refused**: Ensure backend services (stigman, keycloak) are running
- **ACL not matching**: Check HAProxy logs for ACL evaluation, use `option httplog` for detailed logging
- **SSL errors**: Verify DoD CA certs are properly copied to the container

### Testing ACLs

You can test HAProxy ACLs and configuration syntax:

```bash
docker-compose exec haproxy haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

### Viewing Statistics

HAProxy has a built-in statistics page. To enable it, add to `haproxy.cfg`:

```haproxy
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
```

Then access at http://localhost:8404/stats

## Certificate Management

HAProxy expects the server certificate and private key in a single PEM file. The `combined.pem` file is pre-generated:

```bash
cat certs/localhost/localhost.crt certs/localhost/localhost.key > haproxy/combined.pem
```

If you update certificates, regenerate this file:
```bash
cd stigman-orchestration
cat certs/localhost/localhost.crt certs/localhost/localhost.key > haproxy/combined.pem
```

## Performance Tuning

HAProxy is highly optimized for performance. Key configuration options:

- `maxconn` - Maximum concurrent connections (default: 2000 in this config)
- `tune.ssl.default-dh-param` - DH parameter size for SSL (2048 in this config)
- `timeout connect/client/server` - Connection timeout values

For high-traffic environments, consider:
- Increasing `maxconn`
- Enabling connection reuse with `http-reuse`
- Using `nbthread` for multi-core systems
- Tuning TCP stack parameters

## Security Considerations

This configuration:
- Enforces TLS 1.2+ (HAProxy default)
- Validates client certificates against DoD PKI chain
- Requires valid client certificates for OAuth endpoints
- Uses strong cipher suites (HAProxy defaults)
- Forwards original client IP via X-Forwarded-For

For production:
- Review and customize cipher suites
- Consider HSTS headers
- Implement rate limiting
- Enable logging to external syslog
- Restrict admin socket access
