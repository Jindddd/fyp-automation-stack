# FYP Automation Stack

A comprehensive automation stack combining NetBox and Semaphore for network infrastructure management and Ansible automation.

## Overview

This project provides a containerized solution that integrates:

- **NetBox** - An open-source IP address management (IPAM) and data center infrastructure management (DCIM) tool
- **Semaphore** - A modern UI for Ansible automation that makes it easy to run Ansible playbooks

The integration allows you to manage your network infrastructure in NetBox and automate configuration tasks using Semaphore with Ansible.

## Architecture

```
┌─────────────────────────────────────────┐
│     FYP Automation Stack                │
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────────┐  ┌─────────────────┐ │
│  │   NetBox     │  │   Semaphore     │ │
│  │   (IPAM/     │  │   (Ansible      │ │
│  │    DCIM)     │  │    Automation)  │ │
│  └──────┬───────┘  └────────┬────────┘ │
│         │                    │          │
│         └────────┬───────────┘          │
│                  │                      │
│         ┌────────▼─────────┐            │
│         │    PostgreSQL    │            │
│         │    (Database)    │            │
│         └──────────────────┘            │
└─────────────────────────────────────────┘
```

## Components

### NetBox Docker
- Based on the official [netbox-community/netbox-docker](https://github.com/netbox-community/netbox-docker) project
- Provides containerized NetBox deployment
- Includes PostgreSQL database backend

### Semaphore
- Custom Docker image based on `semaphoreui/semaphore`
- Pre-configured with:
  - Python 3 and pip
  - Ansible
  - NetBox Ansible collection (`netbox.netbox`)
  - Additional Python libraries: `paramiko`, `netaddr`, `pynetbox`, `pytz`
- Uses PostgreSQL as the database backend
- Network mode: host (for easier connectivity)

## Prerequisites

- Docker (version 20.10 or higher)
- Docker Compose (version 2.0 or higher)
- At least 4GB of RAM available
- At least 10GB of free disk space

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/Jindddd/fyp-automation-stack.git
cd fyp-automation-stack
```

### 2. Configure NetBox

Navigate to the NetBox directory and set up the environment:

```bash
cd netbox-docker
```

Create a `docker-compose.override.yml` file to expose NetBox:

```bash
tee docker-compose.override.yml <<EOF
services:
  netbox:
    ports:
      - 8000:8080
EOF
```

For additional configuration options, refer to the [NetBox Docker documentation](./netbox-docker/README.md).

### 3. Configure Semaphore

Navigate to the Semaphore directory:

```bash
cd ../semaphore
```

Create a `.env` file with your configuration:

```bash
cat > .env <<EOF
DB_USER=semaphore
DB_PASSWORD=semaphore
DB_HOST=postgres
DB_PORT=5432
ADMIN_PASSWORD=changeme
EOF
```

**Important:** Change the `ADMIN_PASSWORD` to a secure password before deployment.

### 4. Start the Services

#### Start NetBox

```bash
cd ../netbox-docker
docker compose pull
docker compose up -d
```

NetBox will be available at: `http://localhost:8000`

Default credentials:
- Username: `admin`
- Password: `admin`

#### Start Semaphore

```bash
cd ../semaphore
docker compose up -d
```

Semaphore will be available at: `http://localhost:3000`

Default credentials:
- Username: `admin`
- Email: `admin@localhost`
- Password: The value you set in `ADMIN_PASSWORD`

## Usage

### NetBox

1. Access NetBox at `http://localhost:8000`
2. Log in with the default credentials
3. Change the admin password immediately after first login
4. Start adding your network infrastructure:
   - Sites
   - Racks
   - Devices
   - IP addresses
   - VLANs
   - Circuits
   - And more...

For detailed NetBox usage, see the [official NetBox documentation](https://docs.netbox.dev/).

### Semaphore

1. Access Semaphore at `http://localhost:3000`
2. Log in with your admin credentials
3. Create a new project
4. Add your inventory (can be synced from NetBox using `pynetbox`)
5. Create or import Ansible playbooks
6. Run automation tasks

The Semaphore container comes pre-installed with the `netbox.netbox` Ansible collection, allowing you to:
- Query NetBox inventory
- Create/update NetBox objects via Ansible
- Use NetBox as a dynamic inventory source

### Integration Example

Example Ansible playbook to query NetBox from Semaphore:

```yaml
---
- name: Get devices from NetBox
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Query NetBox for devices
      netbox.netbox.nb_lookup:
        api_endpoint: "http://localhost:8000"
        api_version: "3.0"
        token: "your-netbox-api-token"
        plugin: nb_lookup
        api_filter: "name__ic=router"
```

## Configuration

### Environment Variables

#### Semaphore

The following environment variables can be configured in `semaphore/.env`:

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_USER` | PostgreSQL database username | `semaphore` |
| `DB_PASSWORD` | PostgreSQL database password | `semaphore` |
| `DB_HOST` | Database service name | `postgres` |
| `DB_PORT` | PostgreSQL port | `5432` |
| `ADMIN_PASSWORD` | Semaphore admin password | (Required) |

#### NetBox

NetBox configuration is managed through the `netbox-docker/docker-compose.override.yml` file. See the [NetBox Docker documentation](./netbox-docker/README.md) for detailed configuration options.

### Persistent Data

Data is persisted using Docker volumes:

- **NetBox**: PostgreSQL data is stored in volumes managed by the netbox-docker compose file
- **Semaphore**: PostgreSQL data is stored in the `semaphore-postgres` volume

To backup data, you can use `docker volume` commands or backup the database directly.

## Troubleshooting

### Port Conflicts

If you encounter port conflicts:

- NetBox: Change the port mapping in `netbox-docker/docker-compose.override.yml`
- Semaphore: Uses host networking, so ports 3000 and 5432 must be available

### Container Issues

View logs for debugging:

```bash
# NetBox logs
cd netbox-docker
docker compose logs -f netbox

# Semaphore logs
cd semaphore
docker compose logs -f semaphore
```

### Database Connection Issues

If Semaphore cannot connect to the database:

1. Ensure the PostgreSQL container is running: `docker compose ps`
2. Check database credentials in `.env`
3. Verify the `DB_HOST` matches the service name in `docker-compose.yml`

### Resetting Everything

To completely reset the stack:

```bash
# Stop and remove all containers, networks, and volumes
cd netbox-docker
docker compose down -v

cd ../semaphore
docker compose down -v
```

**Warning:** This will delete all data!

## Development

### Building Custom Images

To rebuild the Semaphore image with additional dependencies:

1. Edit `semaphore/Dockerfile`
2. Rebuild: `cd semaphore && docker compose build`
3. Restart: `docker compose up -d`

### Adding Ansible Collections

To add more Ansible collections to Semaphore, edit `semaphore/Dockerfile` and add:

```dockerfile
RUN ansible-galaxy collection install <collection-name>
```

## Project Structure

```
fyp-automation-stack/
├── netbox-docker/          # NetBox Docker deployment
│   ├── docker-compose.yml  # NetBox services configuration
│   ├── Dockerfile          # NetBox container image
│   └── README.md           # NetBox-specific documentation
├── semaphore/              # Semaphore deployment
│   ├── Dockerfile          # Custom Semaphore image with Ansible
│   └── docker-compose.yml  # Semaphore services configuration
├── .gitignore              # Git ignore rules
└── README.md               # This file
```

## Security Considerations

1. **Change Default Passwords**: Always change default passwords in production
2. **API Tokens**: Secure your NetBox API tokens
3. **Network Exposure**: By default, services are exposed locally. Use reverse proxy with SSL for production
4. **Secrets Management**: Use environment variables or secrets management tools for sensitive data
5. **Access Encryption**: The `SEMAPHORE_ACCESS_KEY_ENCRYPTION` should be changed for production deployments

## Contributing

This is a Final Year Project (FYP) repository. If you'd like to contribute:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

This project uses components with different licenses:

- NetBox Docker: Licensed under Apache License 2.0
- Semaphore: Licensed under MIT License

Please refer to each component's license file for details.

## Resources

- [NetBox Documentation](https://docs.netbox.dev/)
- [NetBox Docker Repository](https://github.com/netbox-community/netbox-docker)
- [Semaphore Documentation](https://docs.ansible-semaphore.com/)
- [Ansible Documentation](https://docs.ansible.com/)
- [NetBox Ansible Collection](https://docs.ansible.com/ansible/latest/collections/netbox/netbox/)

## Support

For issues related to:
- **NetBox Docker**: See the [netbox-docker repository](https://github.com/netbox-community/netbox-docker)
- **Semaphore**: See the [Semaphore documentation](https://docs.ansible-semaphore.com/)
- **This Integration**: Open an issue in this repository

## Acknowledgments

- NetBox Community for the excellent IPAM/DCIM tool
- Semaphore UI team for the Ansible automation interface
- All contributors to the open-source tools used in this stack
