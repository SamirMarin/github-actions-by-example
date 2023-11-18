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

Right now this will simply print out the workout that was created to the console. So not a whole lot is really happening here. Other than we can validate that the post request body we are passing in our curl is getting deserialize into a workout struct.
Next up we will add logic to save the workout to a database.

#### Adding the workout to a database
For our database we will use Amazon local dynamodb, Amazon dynamodb a popular NoSql db used for microservices when running in AWS cloud. We will therefore use dynamodb local to simulate a dynamodb instance running in the cloud. Using local dynamodb is a common practice for local development. It will also allow us to mic the dynamodb instance in our github action test workflow.

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

