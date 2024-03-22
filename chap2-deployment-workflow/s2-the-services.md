# The services
To illustrate the deployment workflow, we will develop two basic Golang HTTP services: the [workout-management-service](https://github.com/SamirMarin/workout-management-service) and the [user-management-service](https://github.com/SamirMarin/user-management-service).

Both services have similar implementation details; therefore, we will focus primarily on the implementation of one service, the workout-management-service.

## The workout-management-service
This service is a straightforward HTTP service providing a REST API to manage workouts. Its primary functions include creating workouts and querying the created workouts.

### Creating the golang project

We begin by setting up our Golang project. The first step is to create a project directory and initialize a Go module within it.

```bash
# create directory
mkdir workout-management-service

# here, we'll use 'github.com/samirmarin/workout-management-service' as the module name
go mod init github.com/samirmarin/workout-management-service
```

In this instance, the module name incorporates my GitHub account and the repository name. This naming convention is particularly relevant for importing packages from this project into others. However, since we won’t be doing such imports in this project, feel free to choose any module name you prefer.

Executing this command will generate a go.mod file in the project's root directory. Additionally, a go.sum file will also be created as we start importing external packages into our project.

### The http server
For our HTTP server framework, we will utilize [Echo](https://echo.labstack.com/). To install Echo, execute the following command from the root directory of our project:

```bash
go get github.com/labstack/echo/v4
```

Next, create a main.go file in the root directory of our project. Insert the code below to set up a basic HTTP server that listens on port 1323:

```go
package main

import (
	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()
	e.Logger.Fatal(e.Start(":1323"))
}
```

To build and run this server, use the following commands from the root of our project:

```bash
go build -o workout-management-service
./workout-management-service
```

Executing these commands will activate the server on port 1323. To test the server, you can use the command:

```bash
curl localhost:1323
```
Since we haven’t configured any routes yet, this should return a 404 response.

### Adding routes

#### The create route
Let's implement a route for creating a workout. We'll set up an endpoint to handle POST requests at /create.

We'll modify our main.go file to include a create function:

```go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()
    e.POST("/create", create)
	e.Logger.Fatal(e.Start(":1323"))
}

func create(c echo.Context) error {
	return c.String(http.StatusOK, "Workout created")
}
```

To test this new route, execute these commands:

```bash
go build -o workout-management-service
./workout-management-service
curl -v -X POST localhost:1323/create
```

This should result in a 200 response, indicating success. Currently, this route simply returns a message, "Workout created."

#### The get route
Adding the get route follows a similar process to the create route. We'll add a route that listens for POST requests at the /get endpoint.

Update main.go as follows:

```go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()
    e.POST("/create", create)
    e.POST("/get", get)
	e.Logger.Fatal(e.Start(":1323"))
}

func create(c echo.Context) error {
	return c.String(http.StatusOK, "Workout created")
}

func get(c echo.Context) error {
	return c.JSON(http.StatusOK, "Getting workout")
}

```
In this case, we use a POST request for the get route to enable passing a request body. This body will contain the owner's name and the name of the workout we wish to retrieve.

To test this route, use the following commands:

```bash
go build -o workout-management-service
./workout-management-service
curl -v -X POST localhost:1323/get
```

This will also return a 200 response, but like the create route, it currently only prints out "Getting workout."

### Creating a workout pkg
Let's develop a package responsible for workout creation and retrieval. We'll name this package workout and include a functionx for creating workouts and getting workouts.

First, set up the necessary directory structure and files:

```bash
mkdir -p internal/workout
touch internal/workout/workout.go
```

#### Defining the Workout Structure
Start by defining the Workout struct in workout.go:

```go
package workout

type Workout struct {
	Owner     string     `json:"owner"`
	Name      string     `json:"name"`
	Category  string     `json:"category"`
	Equipment Equipment  `json:"equipment"`
	Exercises []Exercise `json:"exercises"`
}

type Equipment struct {
	Name        string `json:"name"`
	Description string `json:"description"`
}

type Exercise struct {
	Name        string `json:"name"`
	Description string `json:"description"`
	Sets        int    `json:"reps"`
	Time        int    `json:"time"`
}

```

#### Adding Create and Get Functions

Now, add the CreateWorkout and GetWorkout functions to handle creating and querying workouts:

```go
package workout

import "fmt"

type Workout struct {
	Owner     string     `json:"owner"`
	Name      string     `json:"name"`
	Category  string     `json:"category"`
	Equipment Equipment  `json:"equipment"`
	Exercises []Exercise `json:"exercises"`
}

type Equipment struct {
	Name        string `json:"name"`
	Description string `json:"description"`
}

type Exercise struct {
	Name        string `json:"name"`
	Description string `json:"description"`
	Sets        int    `json:"reps"`
	Time        int    `json:"time"`
}

// CreateWorkout creates a workout, save new workout to db
func (w *Workout) CreateWorkout() error {
	fmt.Println(w)
	return nil
}

// GetWorkout gets a workout from db
func (w *Workout) GetWorkout() error {
	fmt.Println(w)
	return nil
}

```

#### Integrating with with our routes
In the main package, update route handling to use these new functions. We'll use echo.Context to parse the request body into a Workout struct:

```go

package main

import (
	"net/http"

	"github.com/SamirMarin/test-golang-commands/internal/workout"
	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()
	e.POST("/create", create)
	e.POST("/get", create)
	e.Logger.Fatal(e.Start(":1323"))
}

func create(c echo.Context) error {
	workout := workout.Workout{}
	if err := c.Bind(&workout); err != nil {
		return echo.NewHTTPError(http.StatusBadRequest, err.Error())
	}
	err := workout.CreateWorkout()
	if err != nil {
		return echo.NewHTTPError(http.StatusBadRequest, err.Error())
	}

	return c.String(http.StatusOK, "Workout created")
}

func get(c echo.Context) error {
	workout := workout.Workout{}
	if err := c.Bind(&workout); err != nil {
		return echo.NewHTTPError(http.StatusBadRequest, err.Error())
	}
	err := workout.GetWorkout()
	if err != nil {
		return echo.NewHTTPError(http.StatusBadRequest, err.Error())
	}

	return c.JSON(http.StatusOK, workout)
}

```

#### Testing the Routes

You can test these routes with the following curl commands:

first rebuild and run the service:

```bash
go build -o workout-management-service
./workout-management-service
```

then run the follwoing curl commands:

```bash
curl -X POST http://localhost:1323/create \
-H "Content-Type: application/json" \
-d '{
  "owner": "samir@gmail.com",
  "name": "Run The Interval",
  "category": "running",
  "equipment": {
    "name": "Running Equipment",
    "description": "If indoors we need a threadmill, if outdoors a good place to run fast for 3 min without interruption"
  },
  "exercises": [
    {
      "name": "Warmup",
      "description": "20min jog"
    },
    {
      "name": "3 min by 5 interval",
      "description": "3min x5 at 5k pace with 1min jog"
    },
    {
      "name": "Cooldown",
      "description": "20min cool down"
    }
  ]
}'
```
```bash
curl -X POST http://localhost:1323/create \
-H "Content-Type: application/json" \
-d '{
  "owner": "samir@gmail.com",
  "name": "Run The Interval",
 }'
```

Currently, these routes will simply print the deserialized Workout structs to the console. This allows us to verify that the POST request bodies are being correctly deserialized into Workout structs.

### The Database
For our database, we will use Amazon DynamoDB Local, a popular NoSQL database choice for microservices in AWS cloud environments. DynamoDB Local simulates a DynamoDB instance in the cloud, making it an ideal choice for local development and testing in GitHub Actions workflows.

We choose DynamoDB due to its widespread use in industry, particularly for microservices hosted on AWS. However, it's worth noting that other databases could be substituted depending on specific requirements or preferences.

#### dynamodb package

We'll start by creating a DynamoDB package in our internal directory. This involves setting up a dynamodb.go file in the internal/dynamodb directory. This package will initially be used in both services, but later we'll explore how to extract it into its own repository for better reusability.

##### DynamoDB Client
Here's the initial setup for our DynamoDB client:

```golang
package dynamodb

import (
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
	"os"
)

type Client struct {
	Dynamodb  *dynamodb.DynamoDB
	TableName string
}

func NewClient(tableName string) *Client {
	sess := session.Must(session.NewSessionWithOptions(session.Options{
		SharedConfigState: session.SharedConfigEnable,
	}))
	// Optional: Override with local endpoint if an environment variable is set
	var svc *dynamodb.DynamoDB
	if localEndpoint := os.Getenv("DYNAMODB_LOCAL_ENDPOINT"); localEndpoint != "" {
		svc = dynamodb.New(sess, &aws.Config{
			Endpoint: aws.String(localEndpoint),
			Region:   aws.String("us-west-2"),
			// provide test credentials when connecting to DynamoDB local
			Credentials: credentials.NewStaticCredentials("test", "test", ""),
			// Disable SSL for local non-production use
			DisableSSL: aws.Bool(true),
		})
	} else {
		svc = dynamodb.New(sess)
	}

	return &Client{
		Dynamodb:  svc,
		TableName: tableName,
	}
}

```
Our client is designed to work with both local and cloud instances of DynamoDB. By default, it connects to a local instance if the DYNAMODB_LOCAL_ENDPOINT environment variable is set, or to a cloud instance based on AWS credentials otherwise.

##### Storable Interface and Functions
Let's define a Storable interface and add StoreItem and GetItem functions to interact with DynamoDB:

```golang
...

type Storable interface {
	ToDynamoDbAttribute() map[string]*dynamodb.AttributeValue
	ToDynamoDbItemInput() *dynamodb.GetItemInput
}

func (c *Client) StoreItem(itemToStore Storable) error {
	item := itemToStore.ToDynamoDbAttribute()

	_, err := c.Dynamodb.PutItem(&dynamodb.PutItemInput{
		Item:      item,
		TableName: aws.String(c.TableName),
	})
	if err != nil {
		return err
	}

	return nil
}

func (c *Client) GetItem(itemToSearch Storable) (error, *dynamodb.GetItemOutput) {
	item := itemToSearch.ToDynamoDbItemInput()
	itemOutput, err := c.Dynamodb.GetItem(item)
	if err != nil {
		return err, nil
	}

	return nil, itemOutput
}

```
We use the Storable interface to keep our functions generic and reusable. This allows our Workout struct to implement methods that convert it to DynamoDB-compatible formats.

##### Integrating with Workout pkg
Next, integrate these methods with our Workout pkg:
Add the ToDynamoDbAttribute and ToDynamoDbItemInput functions to our workout pkg. This ensures that our workout pkg implements the Storable interface.

```golang
package workout

import (
	"github.com/SamirMarin/workout-management-service/internal/dynamodb"
	"github.com/aws/aws-sdk-go/aws"
	awsDynamoDb "github.com/aws/aws-sdk-go/service/dynamodb"
	awsDynamoDbAttribute "github.com/aws/aws-sdk-go/service/dynamodb/dynamodbattribute"
	"strconv"
)

...
func (w *Workout) ToDynamoDbAttribute() map[string]*awsDynamoDb.AttributeValue {
	exerciseList := make([]*awsDynamoDb.AttributeValue, len(w.Exercises))
	for i, exercise := range w.Exercises {
		exerciseList[i] = &awsDynamoDb.AttributeValue{
			M: map[string]*awsDynamoDb.AttributeValue{
				"Name": {
					S: aws.String(exercise.Name),
				},
				"Description": {
					S: aws.String(exercise.Description),
				},
				"Sets": {
					N: aws.String(strconv.Itoa(exercise.Sets)),
				},
				"Time": {
					N: aws.String(strconv.Itoa(exercise.Time)),
				},
			},
		}
	}
	return map[string]*awsDynamoDb.AttributeValue{
		"Owner": {
			S: aws.String(w.Owner),
		},
		"Name": {
			S: aws.String(w.Name),
		},
		"Category": {
			S: aws.String(w.Category),
		},
		"Equipment": {
			M: map[string]*awsDynamoDb.AttributeValue{
				"name": {
					S: aws.String(w.Equipment.Name),
				},
				"description": {
					S: aws.String(w.Equipment.Description),
				},
			},
		},
		"Exercises": {
			L: exerciseList,
		},
	}
}

func (w *Workout) ToDynamoDbItemInput() *awsDynamoDb.GetItemInput {
	return &awsDynamoDb.GetItemInput{
		TableName: aws.String(tableName),
		Key: map[string]*awsDynamoDb.AttributeValue{
			"Owner": {
				S: aws.String(w.Owner),
			},
			"Name": {
				S: aws.String(w.Name),
			},
		},
	}
}

```

These methods ensure that our Workout struct can be stored and retrieved from DynamoDB. By converting our workout struct to a dynamodb attribute and a dynamodb item input. 
- the dynamodb attribute is what we will use to store our workout in the database 
- the dynamodb item input is what we will use to search for our workout in the database.

##### Modifying Workout Functions
Update the CreateWorkout and GetWorkout functions in the workout package to interact with the database:

```golang
...
var tableName = "Workout"

// CreateWorkout creates a workout, save new workout to db
func (w *Workout) CreateWorkout() error {
	dynamoDbClient := dynamodb.NewClient(tableName)
	err := dynamoDbClient.StoreItem(w)
	if err != nil {
		return err
	}
	return nil
}
func (w *Workout) GetWorkout() error {
	dynamoDbClient := dynamodb.NewClient(tableName)
	err, getItemOutput := dynamoDbClient.GetItem(w)
	if err != nil {
		return err
	}
	err = awsDynamoDbAttribute.UnmarshalMap(getItemOutput.Item, w)
	return nil
}
```

With these modifications, our application can now create and retrieve workouts from the database.

#### Setting Up DynamoDB Local

To run DynamoDB locally, we'll use Docker Compose:

```bash
touch docker-compose.yaml
```
Add the following configuration to docker-compose.yaml:

```yaml
version: '3.8'
services:
  dynamodb-local:
    command: "-jar DynamoDBLocal.jar ${DOCKER_COMPOSE_COMMAND_OPTS:-sharedDb -dbPath ./data}"
    image: "amazon/dynamodb-local:latest"
    container_name: dynamodb-local
    ports:
      - "8000:8000"
    volumes:
      - "./docker/dynamodb:/home/dynamodblocal/data"
    working_dir: /home/dynamodblocal
```

Start the local DynamoDB instance:

```bash
docker-compose up -d
```

This will launch a DynamoDB instance on port 8000.

#### Creating a DynamoDB Table

Create a create-table.sh script to set up the DynamoDB table:

```bash
mkdir -p scripts/dynamodb
touch scripts/dynamodb/create-table.sh
```

Add the following script to create the Workout table:
```bash
#!/bin/bash

# Command to create a DynamoDB table
# Specifying ReadCapacityUnits and WriteCapacityUnits is required in local mode
aws dynamodb create-table \
    --table-name Workout \
    --attribute-definitions \
        AttributeName=Owner,AttributeType=S \
        AttributeName=Name,AttributeType=S \
    --key-schema \
        AttributeName=Owner,KeyType=HASH \
        AttributeName=Name,KeyType=RANGE \
    --provisioned-throughput \
        ReadCapacityUnits=5,WriteCapacityUnits=5 \
    --table-class STANDARD \
    --region us-west-2 \
    --endpoint-url http://localhost:8000
```

Execute the script to create the table:
```bash
chmod +x scripts/dynamodb/create-table.sh
./scripts/dynamodb/create-table.sh
```

This script sets up a Workout table with Owner and Name as the primary key components. The Owner attribute serves as the partition key, and Name as the sort key, allowing multiple workouts per owner with unique names.
For more details on dynamodb primary keys see [here](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.PrimaryKey).

### Testing the Create and Get Routes End to End
Now that we have integrated our service with the DynamoDB database, it's time to test the create and get workout functionalities.

#### Rebuilding and Running the Service
First, rebuild and run the service:

```bash
go build -o workout-management-service
./workout-management-service
```
#### Testing the Create Route

To create a new workout, execute the following curl command:

```bash
curl -X POST http://localhost:1323/create \
-H "Content-Type: application/json" \
-d '{
  "owner": "samir@gmail.com",
  "name": "Run The Interval",
  "category": "running",
  "equipment": {
    "name": "Running Equipment",
    "description": "If indoors we need a threadmill, if outdoors a good place to run fast for 3 min without interruption"
  },
  "exercises": [
    {
      "name": "Warmup",
      "description": "20min jog"
    },
    {
      "name": "3 min by 5 interval",
      "description": "3min x5 at 5k pace with 1min jog"
    },
    {
      "name": "Cooldown",
      "description": "20min cool down"
    }
  ]
}'
```
This command should now store the workout details in the database.

#### Testing the Get Route
To retrieve the workout you just created, use the following curl command:

```bash
curl -X POST http://localhost:1323/create \
-H "Content-Type: application/json" \
-d '{
  "owner": "samir@gmail.com",
  "name": "Run The Interval",
 }'
```

This request should return the details of the "Run The Interval" workout from the database.

#### Unit tests
Our services need to be equipped with a set of tests that can be executed with the go test. To demonstrate this, we will craft a few basic unit tests.

We will focus our unit testing on four key functions:
- ToDynamoDbAttribute
- ToDynamoDbItemInput
- CreateWorkout
- GetWorkout

While covering these functions does not exhaustively test all functionality of the service, it provides a fundamental level of coverage. These tests serve as a representative sample to illustrate the process of unit testing.

Reference [workout_test.go](https://github.com/SamirMarin/workout-management-service/blob/main/internal/workout/workout_test.go) for implementation details.

All tests will utilize Go's standard testing package. The core principle of each test is to validate the expected output of a function or series of functions. This pattern will be consistently applied across all unit tests.

##### ToDynamoDbAttribute unit test
For the ToDynamoDbAttribute unit test, we will invoke the function with a predefined Workout struct and verify that the output aligns with our expectations:

```go
func TestToDynamoDbAttribute(t *testing.T) {
	workout := &Workout{
		// ... Initialize workout struct
	}
	dynamodbAttribute := workout.ToDynamoDbAttribute()

	expectedDynamodbAttribute := map[string]*awsDynamoDb.AttributeValue{
		// ... Expected DynamoDB attribute map
	}

	if !reflect.DeepEqual(dynamodbAttribute, expectedDynamodbAttribute) {
		t.Errorf("ToDynamoDbAttribute() = %v, want %v", dynamodbAttribute, expectedDynamodbAttribute)
	}
}
```

##### ToDynamoDbItemInput unit
The ToDynamoDbItemInput test follows a similar structure. We'll invoke ToDynamoDbItemInput and validate that the function's output matches our expected result.

##### CreateWorkout and GetWorkout unit test
The CreateWorkout and GetWorkout functions interact with the DynamoDB instance, requiring a running local instance of DynamoDB (facilitated by Docker Compose). These tests are be designed to:

1. Use CreateWorkout to store a Workout in DynamoDB, verifying that no errors occur.
2. Retrieve the same Workout using GetWorkout, ensuring the fetched data matches the stored data.

This process not only tests the functionality of each method but also validates their interaction with the database.

##### Running the unit test
To execute the unit tests, ensure that the local DynamoDB instance is active and the necessary tables are set up. Run the tests using the following command in the terminal:

```yaml
go test -v ./...
```

## The user-management-service
The user-management-service, much like the workout-management-service, is an HTTP service that provides a REST API for managing user data. Its core functionalities include creating user profiles and retrieving information about existing users.

Given that the implementation of the user-management-service closely mirrors that of the workout-management-service, we will not repeat the detailed discussion here. Instead, if you are interested in understanding its implementation specifics, you are encouraged to refer to the dedicated [user-management-service](https://github.com/SamirMarin/user-management-service) repo.


