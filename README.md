* https://www.alexedwards.net/blog/serverless-api-with-go-and-aws-lambda

* Initial setup of role
```
aws iam create-role --role-name lambda-books-executor --assume-role-policy-document file://./trust-policy.json --profile admin

aws iam attach-role-policy --role-name lambda-books-executor --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole --profile admin
```

* Build and prep code
```
env GOOS=linux GOARCH=amd64 go build -o /tmp/main
zip -j /tmp/main.zip /tmp/main
```

* Deploy
```
ACCOUNT_ID=....
aws lambda create-function --function-name books --runtime go1.x \
--role arn:aws:iam::${ACCOUNT_ID}:role/lambda-books-executor \
--handler main --zip-file fileb:///tmp/main.zip
```

* Test
```
# aws lambda invoke --function-name books /tmp/output.json && cat /tmp/output.json
aws lambda invoke --function-name books /dev/stdout
```

* Create DynamoDB
```
aws dynamodb create-table --table-name Books \
--attribute-definitions AttributeName=ISBN,AttributeType=S \
--key-schema AttributeName=ISBN,KeyType=HASH \
--provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

* Provision some data
```
aws dynamodb put-item --table-name Books --item '{"ISBN": {"S": "978-1420931693"}, "Title": {"S": "The Republic"}, "Author":  {"S": "Plato"}}'
aws dynamodb put-item --table-name Books --item '{"ISBN": {"S": "978-0486298238"}, "Title": {"S": "Meditations"},  "Author":  {"S": "Marcus Aurelius"}}'
```


AWS sessions and profiles
---------------------------
* https://aws.amazon.com/premiumsupport/knowledge-center/authenticate-mfa-cli/


Use MFA for Console Only
---------------------------
* https://stackoverflow.com/questions/28177505/enforce-mfa-for-aws-console-login-but-not-for-api-calls/41050898#41050898?newreg=138803454f2c4298ad70b570139f7923


# Yubikey (NOT WORKING ON CHROMEBOOK)
---------------------------
* https://developers.yubico.com/yubikey-personalization/
* https://developers.yubico.com/yubikey-manager/

```
sudo apt install libyubikey-dev
sudo apt install pkg-config
sudo apt install libusb-1.0-0-dev libusb-dev libjson-c-dev
pip install yubikey-manager
```
