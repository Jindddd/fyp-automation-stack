FROM semaphoreui/semaphore:latest

USER root
RUN apk add --no-cache py3-pip
RUN pip install --no-cache --upgrade ansible paramiko netaddr pynetbox pytz
RUN ansible-galaxy collection install netbox.netbox
USER semaphore
