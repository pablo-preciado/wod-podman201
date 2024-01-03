# To do list

 - Change the Workflow section in the "ReadMeFirst" document once all the other modules are defined
 - List of planned modules: networking, storage, managing pods, transitioning to k8s, creating a systemd unit
 - Add at least a video to Podman Desktop because it's super useful

# Our application

podman network create database
podman network create payment
podman network ls

podman run -d --rm --name database --network database quay.io/skupper/patient-portal-database
sleep 100
podman run -d --rm --name payment-processor --network payment quay.io/skupper/patient-portal-payment-processor
podman run -d --rm --name frontend --network payment,database -p 8080:8080 \
-e DATABASE_SERVICE_HOST="database" \
-e DATABASE_SERVICE_PORT="5432" \
-e PAYMENT_PROCESSOR_SERVICE_HOST="payment-processor" \
-e PAYMENT_PROCESSOR_SERVICE_PORT="8080" \
quay.io/skupper/patient-portal-frontend

podman stop --all
podman network prune -f

# Adding volume to our app

List all of the databases: podman exec -it database psql -U patient_portal -d patient_portal -c '\list'

podman volume create patient-portal-data

podman run -d --rm --name database --network database -v patient-portal-data:/var/lib/postgresql/data quay.io/skupper/patient-portal-database
sleep 100
podman run -d --rm --name payment-processor --network payment quay.io/skupper/patient-portal-payment-processor
podman run -d --rm --name frontend --network payment,database -p 8080:8080 \
-e DATABASE_SERVICE_HOST="127.0.0.1" \
-e DATABASE_SERVICE_PORT="5432" \
-e PAYMENT_PROCESSOR_SERVICE_HOST="payment-processor" \
-e PAYMENT_PROCESSOR_SERVICE_PORT="8080" \
quay.io/skupper/patient-portal-frontend

Check the columns in the "appointment_requests" table:
podman exec -it database psql -U patient_portal -d patient_portal -c '\d appointment_requests'

Add a column to our table:
podman exec -it database psql -U patient_portal -d patient_portal -c 'ALTER TABLE appointment_requests ADD workshop INTEGER'

# With the pod

podman network create patient-portal-net
podman volume create patient-portal-data

podman pod create --name frontend-pod

podman run -d --rm --name database --network patient-portal-net -v patient-portal-data:/var/lib/postgresql/data --pod frontend-pod quay.io/skupper/patient-portal-database

podman run -d --rm --name payment-processor --network patient-portal-net quay.io/skupper/patient-portal-payment-processor

podman run -d --rm --name frontend --network patient-portal-net -p 8080:8080 \
-e DATABASE_SERVICE_HOST="localhost" \
-e DATABASE_SERVICE_PORT="5432" \
-e PAYMENT_PROCESSOR_SERVICE_HOST="payment-processor" \
-e PAYMENT_PROCESSOR_SERVICE_PORT="8080" \
quay.io/skupper/patient-portal-frontend



podman volume create patient-portal-data
podman network create patient-portal-net
podman pod create --name frontend-pod --network patient-portal-net

podman run -d --rm --name database -v patient-portal-data:/var/lib/postgresql/data --pod frontend-pod quay.io/skupper/patient-portal-database
sleep 30
podman run -d --rm --name payment-processor --network patient-portal-net quay.io/skupper/patient-portal-payment-processor

podman run -d --rm --name frontend \
-e DATABASE_SERVICE_HOST="0.0.0.0" \
-e DATABASE_SERVICE_PORT="5432" \
-e PAYMENT_PROCESSOR_SERVICE_HOST="payment-processor" \
-e PAYMENT_PROCESSOR_SERVICE_PORT="8080" \
quay.io/skupper/patient-portal-frontend


podman rm --all -f
podman network prune -f
podman volume prune -f 
podman pod prune -f

# Quadlets and our application

Envs work this way: Environment=foo=bar
The rest can be taken from here: https://www.redhat.com/sysadmin/multi-container-application-podman-quadlet

#Create both networks quadlets.

cat << EOF > payment.network
[Network]
EOF
cat << EOF > database.network
[Network]
EOF

#Create volume quadlet:

cat << EOF > patient-portal-data.volume
[Volume]
EOF

#Create the database container quadlet:

cat << EOF > database.container
[Install]
WantedBy=default.target

[Container]
Image=quay.io/skupper/patient-portal-database
ContainerName=database
Volume=patient-portal-data.volume:/var/lib/postgresql/data
Network=database.network
EOF

#Create the payment processor quadlet:

cat << EOF > payment-processor.container
[Install]
WantedBy=default.target

[Unit]
Requires=database.service
After=database.service

[Container]
Image=quay.io/skupper/patient-portal-payment-processor
ContainerName=payment-processor
Network=payment.network
EOF

#Create the frontend quadlet:

cat << EOF > frontend.container
[Install]
WantedBy=default.target

[Unit]
Requires=payment-processor.service
After=payment-processor.service

[Container]
Image=quay.io/skupper/patient-portal-frontend
ContainerName=frontend
Network=payment.network
Network=database.network
Environment=DATABASE_SERVICE_HOST="database"
Environment=DATABASE_SERVICE_PORT="5432"
Environment=PAYMENT_PROCESSOR_SERVICE_HOST="payment-processor"
Environment=PAYMENT_PROCESSOR_SERVICE_PORT="8080"
EOF


mv ./* ~/.config/containers/systemd/
systemctl --user daemon-reload

systemctl --user start database.service
systemctl --user status database.service --no-pager

systemctl --user start payment-processor.service
systemctl --user status payment-processor.service --no-pager

systemctl --user start frontend.service
systemctl --user status frontend.service  --no-pager