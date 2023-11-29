# The services
In order to demonstrate the deployment workflow, we will create two simple golang http services. The workout-management-service and the user-management-service.

The implementation details of both services will be very similar, so we will only cover in detail the implementation of one service: the workout-management-service.

## The workout-management-service
The workout-management-service is a simple http service that exposes a REST API for managing workouts - mainly for creating and querying the workouts created.

## The user-management-service
The user-management-service is a simple http service that exposes a REST API for managing users - mainly for creating and querying the users created.

## The implementation details

### Creating the golang project

Let's start by creating our golang project. To start of, let's create a directory for our project and initialize a go module.

```bash
# create directory
mkdir workout-management-service

# in this cause we will use github.com/samirmarin/workout-management-service as the module name
go mod init github.com/samirmarin/workout-management-service
```

In this case we used my github account along with the reponame as the module name. This is really only important when we want to import pkgs from this project into other projects. We will not be doing this in this case, so you could name the module whatever you want.

Running this command will create a go.mod file in the root of our project, a go.sum file will be also be created when we start importing other pkgs into our project.

### The http server
We will use [echo](https://echo.labstack.com/) as our http server framework. To install echo we will run the following command from the root of our project.

```bash
go get github.com/labstack/echo/v4
```

let's now create a main.go file in the root of our project and add the following code in order to create a simple http server that listens on port 1323.

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

we can build and run this server by running the following commands from the root of our project.

```bash
go build -o workout-management-service
./workout-management-service
```

This will run a server on port 1323. You can test the server by running the following command.

```bash
curl localhost:1323
```
This should return a 404 response, since we have no routes configured.

### Adding a routes

#### The create route
Let's add a route for creating a workout. We will add a route that listens for POST requests on the /create endpoint.

To do this we will add a create function to our main.go file.

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

We can now test this route by running the following command.

```bash
go build -o workout-management-service
./workout-management-service
curl -v -X POST localhost:1323/create
```

This will return a 200 response, but currently this route does nothing other than print out "Workout created" to the console.

#### Creating a workout pkg
Let's create a pkg that will be responsible for creating a workout. We will create a pkg called workout and add a Create Workout function to it.

To this wil will add an internal/workout directory to our project and create a workout.go file in it.

```bash
mkdir -p internal/workout
touch internal/workout/workout.go
```
To start lets create the struct that will represent a workout.

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

We can now add a CreateWorkout function to our workout.go file. For now we will simply have it print out the workout that was created.

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

```

Now that we have defined our workout struct and function we can call with a workout struct, we can now call this function from our create route function in main.
We will use the echo.Context to get the request body and unmarshal it into a workout struct.

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
```

We can now test this route by running the following command.

```bash
go build -o workout-management-service
./workout-management-service
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

Right now this will simply print out the workout that was created to the console. So not a lot is really happening here. Other than we can validate that the post request body we are passing in our curl is getting deserialize into a workout struct.
Next up we will add logic to save the workout to a database.

#### Adding the workout to a database
For our database we will use Amazon local dynamodb, Amazon dynamodb a popular NoSql db used for microservices when running in AWS cloud. We will therefore use dynamodb local to simulate a dynamodb instance running in the cloud. Using local dynamodb is a common practice for local development. It will also allow us to mic the dynamodb instance in our github action test workflow.

In our example we have choosen to use dynamodb because it is a common industry practice to use dynamodb for microservices running in AWS cloud. But another database could easily be used instead.

##### dynamodb package

Lets start of by creating a dynamodb package in our internal directory. We will create a dynamodb.go file in the internal/dynamodb directory. This package will be a copy and paste across both services, in a later section we will show how we can seperate this package into its own repo and import it into both services.

The first thing we will do is add dynamodb client.

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
			// provide dummy credentials when connecting to DynamoDB local
			Credentials: credentials.NewStaticCredentials("dummy", "dummy", ""),
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
For our use case we will only be working the dynamodb local, but we ensure the client is also configure to work with non local dynamodb instances.

What we do here is we create a new dynamodb instance that connects to our local dynamodb instance(we will go over how to setup dynamodb local shortly). If the DYNAMODB_LOCAL_ENDPOINT environment variable is not set, we will connect to dynamodb via the defaults provided by aws credentials configured on the machine running the service.

Once we have our client we can no add functions that will operate on out dynamodb database. In our case we will add a StoreItem and GetItem function.

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

In our case we keep our functions very generic, we use Storable interface to our functions, allowing us to pass the responsibility of converting our workout struct to a dynamodb attribute to the workout struct itself. This allows us to keep our dynamodb package generic and reusable across multiple services.

Lets now add the ToDynamoDbAttribute and ToDynamoDbItemInput functions to our workout struct. This ensures that our workout struct implements the Storable interface.

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

What these two functions do is ensure we can convert our workout struct to a dynamodb attribute and a dynamodb item input. The dynamodb attribute is what we will use to store our workout in the database, and the dynamodb item input is what we will use to search for our workout in the database.

We can now get our CreateWorkout and GetWorkout functions in our workout pkg to store and retrieve workouts from the database instead of simply printing to console.

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

with this in place we can now create a workout and retrieve it from the database. But before we can do this we need to have a dynamodb instance running locally and create a workout table.

##### Running dynamodb local and create table

```bash

To install dynamodb local we will use docker compose. We will create a docker-compose.yaml file in the root of our project and add the following code.
```bash
touch docker-compose.yaml
```

```yaml
version: '3.8'
services:
  dynamodb-local:
    command: "-jar DynamoDBLocal.jar -sharedDb -dbPath ./data"
    image: "amazon/dynamodb-local:latest"
    container_name: dynamodb-local
    ports:
      - "8000:8000"
    volumes:
      - "./docker/dynamodb:/home/dynamodblocal/data"
    working_dir: /home/dynamodblocal
```

To create the local dynamodb instance we will run the following command from the root of our project.

```bash
docker-compose up -d
```

This will create a dynamodb instance running on port 8000. We can now create a workout table in our local dynamodb instance by running a bash script.

lets create a create-table.sh file under scripts/dynamodb directory.

```bash
mkdir -p scripts/dynamodb
touch scripts/dynamodb/create-table.sh
```

In the file we will add the following code.
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

This creates a table called Workout with a hash key of Owner and a range key of Name. We will use the Owner and Name as the primary key for our workout table.
In this case our primary key is composed of partition key and a sort key. The partition key is Owner and the sort key is Name. This means that we can have multiple workouts with the same Owner, but each workout must have a unique Name.
For more details on dynamodb primary keys see [here](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html#HowItWorks.CoreComponents.PrimaryKey).

Run the script to create the table
```bash
chmod +x scripts/dynamodb/create-table.sh
./scripts/dynamodb/create-table.sh
```

#### Testing the create and get routes end to end
We can now test our create and get workout functionality end to end.

Lets rebuild our service and run it.

```bash
go build -o workout-management-service
./workout-management-service
```

We can now create a workout by running the same curl we ran previously.

```bash
go build -o workout-management-service
./workout-management-service
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
expect this time we will actually store the workout in the database.

We can then test the get route by running the following curl.

```bash
go build -o workout-management-service
./workout-management-service
curl -X POST http://localhost:1323/create \
-H "Content-Type: application/json" \
-d '{
  "owner": "samir@gmail.com",
  "name": "Run The Interval",
 }'
```

which will grab the workout we just created from the database.




