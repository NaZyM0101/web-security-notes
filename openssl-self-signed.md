### Create self singed pki in linux 



``````bash
# Set up variables for easier management
export CA_CN="My Redis CA"
export CLIENT_CN="redis-client"
export SERVER_CN="redis-server"

# 1. Create the CA key and certificate
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -days 3650 -out ca.crt -subj "/CN=${CA_CN}"

# 2. Create the server key and signing request
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/CN=${SERVER_CN}"

# 3. Sign the server certificate with the CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -extfile <(cat <<EOF
[v3_req]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF
)

# 4. Create the client key and signing request
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj "/CN=${CLIENT_CN}"

# 5. Sign the client certificate with the CA
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -extfile <(cat <<EOF
[v3_req]
basicConstraints = CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = clientAuth
EOF
)

# 6. Create a PKCS#12 file for the client (containing both cert and key)
openssl pkcs12 -export -out client.p12 -inkey client.key -in client.crt -certfile ca.crt -password pass:your_client_password

# Verify the certificates (Optional)
openssl x509 -in ca.crt -text -noout
openssl x509 -in server.crt -text -noout
openssl x509 -in client.crt -text -noout

#Convert PEM to CRT for redis server if needed
openssl x509 -outform der -in server.crt -out server.crt.der
openssl x509 -outform der -in ca.crt -out ca.crt.der
``````

