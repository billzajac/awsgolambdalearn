* https://www.alexedwards.net/blog/serverless-api-with-go-and-aws-lambda

* Deploy with Ansible
```
ansible-playbook deploy.yml
```

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

* ENV VARIABLES
```
ACCOUNT_ID=....
ZONE=us-east-1
```

* Deploy
```
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

* Rebuild and redeploy
```
env GOOS=linux GOARCH=amd64 go build -o /tmp/main
zip -j /tmp/main.zip /tmp/main
aws lambda update-function-code --function-name books --zip-file fileb:///tmp/main.zip
```

* Add new policy to role for dynamodb access
```
aws iam put-role-policy --role-name lambda-books-executor \
--policy-name dynamodb-item-crud-role \
--policy-document file://./dynamodb-privilege-policy.json --profile admin
```

* Create API gateway
```
aws apigateway create-rest-api --name bookstore
REST_API_ID=....
```

* Get the id of the root API resource ("/")
```
aws apigateway get-resources --rest-api-id ${REST_API_ID}
ROOT_PATH_ID=....
```

* Create /books
```
aws apigateway create-resource --rest-api-id ${REST_API_ID} --parent-id ${ROOT_PATH_ID} --path-part books --profile admin
RESOURCE_ID=....
```

* Configure API gateway to respond to ANY HTTP method
```
aws apigateway put-method --rest-api-id ${REST_API_ID} \
--resource-id ${RESOURCE_ID} --http-method ANY \
--authorization-type NONE
```

* Connect the gateway to proxy with POST to the lambda function
```
aws apigateway put-integration --rest-api-id ${REST_API_ID} \
--resource-id ${RESOURCE_ID} --http-method ANY --type AWS_PROXY \
--integration-http-method POST \
--uri arn:aws:apigateway:${ZONE}:lambda:path/2015-03-31/functions/arn:aws:lambda:${ZONE}:${ACCOUNT_ID}:function:books/invocations
```

* Test API gateway
```
aws apigateway test-invoke-method --rest-api-id ${REST_API_ID} --resource-id ${RESOURCE_ID} --http-method "GET"
```

* Add/fix permissions on 
    * First build a GUID: https://www.guidgenerator.com/
        GUID=....
```
aws lambda add-permission --function-name books --statement-id ${GUID} \
--action lambda:InvokeFunction --principal apigateway.amazonaws.com \
--source-arn arn:aws:execute-api:${ZONE}:${ACCOUNT_ID}:${REST_API_ID}/*/*/*
```

* Test API gateway
```
aws apigateway test-invoke-method --rest-api-id ${REST_API_ID} --resource-id ${RESOURCE_ID} --http-method "GET"
```

* Now fix the response for API gateway from lambda (in the Go code)
```
go get github.com/aws/aws-lambda-go/events

# Update the code to use the new structs
vi books/main.go

cd books && \
env GOOS=linux GOARCH=amd64 go build -o /tmp/main && \
zip -j /tmp/main.zip /tmp/main && \
aws lambda update-function-code --function-name books --zip-file fileb:///tmp/main.zip
```

* Test API gateway
```
aws apigateway test-invoke-method --rest-api-id ${REST_API_ID} --resource-id ${RESOURCE_ID} --http-method "GET" \
--path-with-query-string "/books?isbn=978-1420931693"

# Test book that isn't in the DB
aws apigateway test-invoke-method --rest-api-id ${REST_API_ID} --resource-id ${RESOURCE_ID} --http-method "GET" \
--path-with-query-string "/books?isbn=foobar"
```

* Query Cloudwatch
```
aws logs filter-log-events --log-group-name /aws/lambda/books \
--filter-pattern "ERROR"
```

* Create Deployment
```
aws apigateway create-deployment --rest-api-id ${REST_API_ID} \
--stage-name staging
```

* Test Deployment
```
curl https://${REST_API_ID}.execute-api.us-east-1.amazonaws.com/staging/books?isbn=978-1420931693
curl https://${REST_API_ID}.execute-api.us-east-1.amazonaws.com/staging/books?isbn=foobar

```
