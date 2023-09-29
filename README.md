# DynamoDB Local - Kubernetes Setup Steps

## Clone Helm Chart

```
$ cd ~/Workspace
$ git clone github.com/jittakal/dynamodb-local-helm-chart
```

## Install Helm Chart

```
$ helm install aws ./dynamodb-local-helm-chart
$ kubectl port-forward -n default svc/aws-dynamodb-local 8000:8000
```

## Download & Install NoSQL Workbench

```
$ cd tmp
$ wget -o WorkbenchDDBLocal-mac.zip https://s3.amazonaws.com/nosql-workbench/WorkbenchDDBLocal-mac.zip # Only for Mac
```
Refer [Installation Steps](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/workbench.settingup.install.html)

## Add Localhost Connection

- Go to Operational Builder
- Add Connection for Localhost
-- Host: 127.0.0.1
-- Port: 8000
- Import Sample DynamoDB Tables

Note: DynamoDB Local Running within Kubernetes - Microk8s

## NoSQL Workbench

![NoSQL Workbench](./docs/images/nosql-workbench.png "NoSQL Workbench")

## DynamoDB Local - K8S

![DynamoDB Local K8S](./docs/images/dynamodb-local-k8s.png "DynamoDB Local")

## Using AWS CLI

```
$ # Create Table
$ aws dynamodb create-table --table-name Orders --attribute-definitions AttributeName=orderId,AttributeType=S --key-schema AttributeName=orderId,KeyType=HASH --billing-mode PAY_PER_REQUEST --endpoint-url http://localhost:8000 --region us-east-1
```

```
$ # List Tables
$ aws dynamodb list-tables --endpoint-url http://localhost:8000 --region us-east-1
```

## Using Go

```go
func NewDynamoDBLocalClient(ctx context.Context) (*dynamodb.Client, error) {
	cfg, err := config.LoadDefaultConfig(ctx,
		config.WithRegion("us-east-1"),
		config.WithEndpointResolverWithOptions(aws.EndpointResolverWithOptionsFunc(
			func(service, region string, options ...interface{}) (aws.Endpoint, error) {
				return aws.Endpoint{URL: "http://localhost:8000"}, nil
			})),
		config.WithCredentialsProvider(credentials.StaticCredentialsProvider{
			Value: aws.Credentials{
				AccessKeyID: "AKID", SecretAccessKey: "SECRET",
				Source: "Hard-coded credentials; values are irrelevant for local DynamoDB",
			},
		}),
	)
	if err != nil {
		slog.Error("Error While getting connection")
		return nil, err
	}

	return dynamodb.NewFromConfig(cfg), nil
}
```

```go
func TestListTables(t *testing.T) {
	ctx := context.Background()

	client, err := NewDynamoDBLocalClient(ctx)

	if err != nil {
		t.Errorf("Error while creating local dynamodb connection - %s", err)
	}

	listTableOutput, err := client.ListTables(ctx, &dynamodb.ListTablesInput{
		ExclusiveStartTableName: aws.String("Order"),
	})

	if err != nil {
		t.Errorf("Error while listing tables from local dynamodb connection - %s", err)
	}

	if len(listTableOutput.TableNames) == 0 {
		t.Errorf("List of tables should not be empty")
	}
}
```