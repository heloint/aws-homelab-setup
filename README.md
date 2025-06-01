AWS homelab setup
=================
Basic AWS homelab setup with Localstack, awscli-local and terraform-local.

# Installations

1. Setup venv
```sh
deactivate  # If there's any python virtual environmnent already activated.
python3 -m venv ls-venv
source ./ls-venv/bin/activate
pip install -r requirements.txt
```

2. Start localstack
```sh
localstack start -d
localstack status services
```


# Run the lambda + apigateway example
```sh
tflocal apply -auto-approve
curl http://localhost:4566/restapis/$(awslocal apigateway get-rest-apis --query 'items[0].id' --output text)/dev/_user_request_/hello
```
