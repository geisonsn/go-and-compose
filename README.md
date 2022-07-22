# [Develop a Go app with Docker Compose](https://firehydrant.com/blog/develop-a-go-app-with-docker-compose/)
Learn how to structure a Go application with Docker Compose as your development environment.
_By Robert Ross_ - August 29th, 2021

Writing Go applications in an isolated environment with Docker comes with some great advantages. You get the bare essentials for developing, and you can easily change which Go version you’re developing against.

In this tutorial, we’re going to show you how to structure a Go application with Docker Compose as your development environment.

In the end you'll have:

1. A docker compose setup to develop in
2. An HTTP server written in Go that is connected to Postgres
3. An auto-reloading server that compiles when you change a file

### Technology used
- Go 1.16
- Docker 20.10.7
- [Air](https://github.com/cosmtrek/air) (for live reloading)
- [Migrate](https://github.com/golang-migrate/migrate)
- [gorilla/mux](https://github.com/gorilla/mux)
 
**Note**: compose has been merged into the docker CLI, so we're going to be using docker compose instead of the standalone CLI docker-compose in this guide.


## Getting started

To get started, we're going to create a folder called go-and-compose.

```sh
$ mkdir go-and-copose
$ cd go-and-compose
```

Open this folder in your editor of choice, and we'll get started setting up our development environment.

### Our Dockerfile

Using Docker Compose with Go can be a bit tricky because Go needs to build a binary to run. For a production deployment, our container doesn't (and shouldn't) have all of the individual Go files, it should just have our single binary.

In the past, I've solved this problem by having multiple Dockerfile's scattered in a repo, typically in the format of Dockerfile-dev or Dockerfile-test. This isn't necessary, however, if we use of multi-stage builds in Docker. Docker Compose can also take advantage of multi-stage builds when starting containers.

We're going to eventually have **4 stages** in our Dockerfile, but for now, let's get started with our base and dev stages.

Create a file called Dockerfile in the root of your project, and let's add these few lines:

```docker
FROM golang:1.16 as base

FROM base as dev

RUN curl -sSfL https://raw.githubusercontent.com/cosmtrek/air/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

WORKDIR /opt/app/api
CMD ["air"]
```

#### What does this do?
```docker
FROM golang:1.16 as base
```

This line instructs Docker to create a stage of our container called base. We're deriving this container off of the official golang container. We have no need to get complicated building our own container with Go in it.

Next, we've added a stage that includes the Air project for live reloading. We'll be mounting our Go project's files to `/opt/app/api`.

```docker
# Create another stage called "dev" that is based off of our "base" stage (so we have golang available to us)
FROM base as dev

# Install the air binary so we get live code-reloading when we save files
RUN curl -sSfL https://raw.githubusercontent.com/cosmtrek/air/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

# Run the air command in the directory where our code will live
WORKDIR /opt/app/api
CMD ["air"]
```

Now, let's build our containers and get Air initialized. To do this, we'll need to add our initial docker-compose.yml file (the whole reason you're here, I presume?).

### Our docker-compose.yml file

Here's the starting point of our compose file, we'll be adding and modifying it later to add database and such. Create a new file called docker-compose.yml in the project root and add this snippet:

```sh
version: "3.9"
services:
  app:
    build:
      dockerfile: Dockerfile
      context: .
      target: dev
    volumes:
      - .:/opt/app/api
```

⚠️There are a few things to callout about this docker-compose.yml so we understand how these gears mesh together.

- The `services.app.build.target` value is set to "dev" - This is the same dev that is in our Dockerfile stages.
- The `services.app.volumes[0]` value is mounting the current directory to the same WORKDIR in our Dockerfile "dev" stage.
These may not seem obvious, but they're necessary to understand and for this project to work. The magic is in the details. 🦄🌈

Let's attempt to build our container using the build command of compose (remember, we're using a new version of Docker that has compose built into it now!)

```sh
$ docker compose build
```

You now should have a new container built that has Go and Air installed into it. So let's create a simple Go program and see if we can get it to recompile on save with Air.

### Initial Code

To get the party started, let's add a simple `main.go` to our root directory of the new project.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  for {
    fmt.Println("Hello World")
    time.Sleep(time.Second * 3)
  }
}
```

This simple program prints "Hello World" every 3 seconds. The reason we're doing this is because we want to see a "long lived" start in our container so we can live reloading work later with Air.

Let's also init our Go module from within the container, run the follow command (_replace USER with your GitHub user please_):

```sh
$ docker compose run --rm app go mod init github.com/USER/go-and-compose
```

We need to create a `.air.toml` file that Air will read as well. Air provides a simple command for us to use, which we can run from inside of our new shiny container built from Docker Compose.

### Air Setup

We should run our Air init from inside of our app container (defined in our compose yaml file). To do this, let's run the follow command:

```sh
$ docker compose run --rm app air init
```

**Note**: I prefer using run with the `--rm` flag because I don't like having a bunch of random containers laying around from commands I've run in them.

You should see output similar to:

```sh
➜  go-and-compose docker compose run --rm app air init

  __    _   ___
 / /\  | | | |_)
/_/--\ |_| |_| \_ 1.27.3, built with Go 1.16.3

.air.toml file created to the current directory with the default settings
```

Because of our volume mount defined in our compose file, we should see the file appear in our local filesystem in our project folder, too.

The default .air.toml config file should be fine for our purposes for now.

## Trying it all out
Ok we're at an exciting part of this guide: we get to see the fruits of our labor start to show results. Let's start up our container and see our project come alive.

```sh
$ docker compose up
```

You'll see docker compose kicking off by starting the app container, which by default will run air in it (as defined in our `Dockerfile` dev stage).

You should also see Hello World being printed every three seconds, exciting right?

What's even more exciting, if you go to main.go and change "Hello World" to "Hello Universe" and save you should see Air automatically see the change, rebuild the binary, and start it anew.

```sh
app_1  | Hello World
app_1  | Hello World
app_1  | main.go has changed
app_1  | building...
app_1  | running...
app_1  | Hello Universe
```

## Checkpoint: A live reloading Go program
Here's where we are at this point:

- A singular `Dockerfile` that contains multiple stages for building and running our Go project
- A `docker-compose.yml` file that has our code mounted inside and starts air in our dev stage of our container
- A simple `main.go` file that prints some text every few seconds.

Next, we're going to get wild and add a web server that is connected to a database. If all you wanted was a setup that can automatically reload code inside of a docker compose environment, you're there! If you want more like interacting with a database in a docker compose environment, then buckle up.

## A simple HTTP Server
Before we jump into building our simple little API server, let's take a look at future state so we can understand why I like to approach things the way I do.

```
apiserver/apiserver.go <- main API server
storage/storage.go <- interface to database
main.go <- main entrypoint
```

I prefer to separate my concerns in the early stages of a project, because copying and pasting, renaming references, etc, is a pain. We're going to approach the rest of this tutorial the way I'd build a real API server.

### Our API server package
Let's start to build a simple API server package that can respond to HTTP requests. Create a folder called apiserver in the root of our project. Then create a file called `apiserver.go` in that new folder.

Add the following to the apiserver.go file you just created:

```go
// apiserver/apiserver.go
package apiserver

import (
  "context"
  "errors"
  "net/http"
  "time"

  "github.com/gorilla/mux"
  "github.com/sirupsen/logrus"
)

var defaultStopTimeout = time.Second * 30

type APIServer struct {
  addr string
}

func NewAPIServer(addr string) (*APIServer, error) {
  if addr == "" {
    return nil, errors.New("addr cannot be blank")
  }

  return &APIServer{
    addr: addr,
  }, nil
}

// Start starts a server with a stop channel
func (s *APIServer) Start(stop <-chan struct{}) error {
  srv := &http.Server{
    Addr:    s.addr,
    Handler: s.router(),
  }

  go func() {
    logrus.WithField("addr", srv.Addr).Info("starting server")
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
      logrus.Fatalf("listen: %s\n", err)
    }
  }()

  <-stop
  ctx, cancel := context.WithTimeout(context.Background(), defaultStopTimeout)
  defer cancel()

  logrus.WithField("timeout", defaultStopTimeout).Info("stopping server")
  return srv.Shutdown(ctx)
}

func (s *APIServer) router() http.Handler {
  router := mux.NewRouter()

  router.HandleFunc("/", s.defaultRoute)
  return router
}

func (s *APIServer) defaultRoute(w http.ResponseWriter, r *http.Request) {
  w.WriteHeader(http.StatusOK)
  w.Write([]byte("Hello World"))
}
```

### Breakdown

Let's breakdown this file by each major section.

```go
package apiserver

import (
  "context"
  "errors"
  "net/http"
  "time"

  "github.com/gorilla/mux"
  "github.com/sirupsen/logrus"
)

var defaultStopTimeout = time.Second * 30

type APIServer struct {
  addr string
}

func NewAPIServer(addr string) (*APIServer, error) {
  if addr == "" {
    return nil, errors.New("addr cannot be blank")
  }

  return &APIServer{
    addr: addr,
  }, nil
}
```

I don't like exposing certain fields on a server since they should never be modified after a server has been started anyways. For example the addr field. I prefer to create a factory method and assign the unexported field there. Also, you'll notice we're using gorilla/mux for our request router as well as logrus for our logging.

So our NewAPIServer function returns an initialized server, now what?

```go
// Start starts a server with a stop channel
func (s *APIServer) Start(stop <-chan struct{}) error {
  srv := &http.Server{
    Addr:    s.addr,
    Handler: s.router(),
  }

  go func() {
    logrus.WithField("addr", srv.Addr).Info("starting server")
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
      logrus.Fatalf("listen: %s\n", err)
    }
  }()

  <-stop
  ctx, cancel := context.WithTimeout(context.Background(), defaultStopTimeout)
  defer cancel()

  logrus.WithField("timeout", defaultStopTimeout).Info("stopping server")
  return srv.Shutdown(ctx)
}
```

I prefer to expose an interface for starting and stopping a server that is simple from the caller. In this case, we're using a stop channel that we'll close from the main.go file when we receive a signal to our process to stop a server from accepting connections (you'll see this soon).

We initialize a new `http.Server{}` with a handler (in our case, `gorilla/mux`) and the address field we initialized our APIServer with.

This method will also block while the server is running. When the stop channel is closed, we'll wait a default of 30 seconds to let the server finish processing any open connections. This is accomplished with the `context.WithTimeout()` and `srv.Shutdown(ctx)` lines.

So where does our server logic go?

```go
func (s *APIServer) router() http.Handler {
  router := mux.NewRouter()

  router.HandleFunc("/", s.defaultRoute)
  return router
}

func (s *APIServer) defaultRoute(w http.ResponseWriter, r *http.Request) {
  w.WriteHeader(http.StatusOK)
  w.Write([]byte("Hello World"))
}
```

Our `router()` method is fairly simple in that it returns an initialized `gorilla/mux` dispatcher that responds to `/` requests. In a more complex project, these would likely be split into other files, or even packages. Our `defaultRoute()` method simply responds with **_"Hello World_"** to the request.

### Using our API server
Our trusty `main.go` file is about to receive a major makeover. We're going to be using `urfave/cli` to make a nice CLI that we use to start our API server.

```go
package main

import (
  "os"
  "os/signal"
  "syscall"

  // Make sure you change this line to match your module
  "github.com/USER/go-and-compose/apiserver"
  "github.com/sirupsen/logrus"
  "github.com/urfave/cli/v2"
)

const (
  apiServerAddrFlagName string = "addr"
)

func main() {
  if err := app().Run(os.Args); err != nil {
    logrus.WithError(err).Fatal("could not run application")
  }
}

func app() *cli.App {
  return &cli.App{
    Name:  "api-server",
    Usage: "The API",
    Commands: []*cli.Command{
      apiServerCmd(),
    },
  }
}

func apiServerCmd() *cli.Command {
  return &cli.Command{
    Name:  "start",
    Usage: "starts the API server",
    Flags: []cli.Flag{
      &cli.StringFlag{Name: apiServerAddrFlagName, EnvVars: []string{"API_SERVER_ADDR"}},
    },
    Action: func(c *cli.Context) error {
      done := make(chan os.Signal, 1)
      signal.Notify(done, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

      stopper := make(chan struct{})
      go func() {
        <-done
        close(stopper)
      }()

      addr := c.String(apiServerAddrFlagName)
      server, err := apiserver.NewAPIServer(addr)
      if err != nil {
        return err
      }

      return server.Start(stopper)
    },
  }
}
```

### Breakdown

The top portion of this file is relatively simple and can be mostly understood by reading the `urfave/cli` **README**. Let's focus on our `apiServerCmd()` function instead.

```go
func apiServerCmd() *cli.Command {
  return &cli.Command{
    Name:  "start",
    Usage: "starts the API server",
    Flags: []cli.Flag{
      &cli.StringFlag{Name: apiServerAddrFlagName, EnvVars: []string{"API_SERVER_ADDR"}},
    },
    Action: func(c *cli.Context) error {
      done := make(chan os.Signal, 1)
      signal.Notify(done, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

      stopper := make(chan struct{})
      go func() {
        <-done
        close(stopper)
      }()

      addr := c.String(apiServerAddrFlagName)
      server, err := apiserver.NewAPIServer(addr)
      if err != nil {
        return err
      }

      return server.Start(stopper)
    },
  }
}
```

This returns a command called `"start"`. This eventually allows us to use our CLI like so when our binary is built:

```sh
$ api-server start --addr :3000
```

**Note**: This won't work yet

Next, this code is assigning an Action field that sets up a channel to receive **SIGINT** and **SIGTERM** signals. This is more or less "transposed" to close a stopper channel that we've created and have passed to the `server.Start(stopper)` call at the end of our action definition.

### Starting our server
We've added a lot of imports to our project, let's run a command inside of our container to tidy up our modules.

```go
$ docker compose run --rm app go mod tidy
```

We need to also update our .air.toml file to use our new subcommand of our CLI (start). Air simply will run the process with no arguments, so let's change that default behavior.

```diff
- full_bin = ""
+ full_bin = "./tmp/main start"
```

This tells air to run the command with start as an argument.

We'll also need to make a small update to our `docker-compose.yml` file to include an environment variable for which address we want to listen on as well as a port exposure. The entirety of the file should look like:

```docker
version: "3.9"
services:
  app:
    build:
      dockerfile: Dockerfile
      context: .
      target: dev
    volumes:
      - .:/opt/app/api
    environment:
      API_SERVER_ADDR: ":3000"
    ports:
    - "3000:3000"
```

#### Summary of changes
- We've updated our main.go file to start a server using our apiserver package
- We've updated our Air config to use the subcommand start
- We've updated our compose YAML to include an environment variable for the server address, and added a port mapping.

## Starting our server up

Now, with all of our changes, we should be able to start up our server again:

```sh
$ docker compose up
```

I see the following as output:

```sh
➜  go-and-compose docker compose up
[+] Running 1/1
 ⠿ Container go-and-compose_app_1  Started 0.6s Attaching to app_1
app_1  | running...
app_1  | INFO[0000] starting server addr=":3000"
```
When I visit`localhost:3000` in my browser I see "Hello World". Isn't it beautiful? 😍

## Finalé: Adding a database

The final frontier of this tutorial is adding a database to our docker compose setup that our API server can utilize for its operations. Here's what's next:

- Add a postgres container
- Add a way to migrate the database
- Create some dummy data in the database each page request (and list it)

### First update to our compose yaml
Let's get going by adding a postgres container to our docker-compose.yml file. We'll be adding another service under the services key.

Let's add:

```docker
  db:
    image: postgres:13-alpine
    volumes:
      - data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: api
      POSTGRES_USER: local-dev
      POSTGRES_HOST_AUTH_METHOD: trust
```

Make sure that it is properly spaced under the services key.

The environment variables are also utilized by the default postgres container to create a database called `"api"` and a user called `"local-dev"` that we can use to connect from our API server here soon. The `POSTGRES_HOST_AUTH_METHOD: trust` portion removes the need for a password to connect.

You may also notice that our volumes key has a `data:` portion, this is referencing a volume that we need to add to our compose config as well.

```docker
volumes:
  data:
```

This may look strange, but it instructs docker compose to create a volume called data. We use this created volume to store our database data into, so when we stop our containers, the information created isn't lost. Volumes are not a part of the services key, they are at the root of our YAML definition.

Lastly, let's link up our app container to our db container by adding this to our services.app definition:

```diff
     ports:
     - "3000:3000"
+    links:
+    - db
```

Let's also reference this new linkage by adding an environment variable to our `services.app.environment` key:

```docker
DATABASE_URL: postgres://local-dev@db/api?sslmode=disable 
```

We'll be modifying our `main.go` file to pass this new value to our server so we can connect and read/write data to our postgres database.

Here's what the `docker-compose.yml` file should look like this in its entirety with these changes:

```docker
version: "3.9"
services:
  app:
    build:
      dockerfile: Dockerfile
      context: .
      target: dev
    volumes:
    - .:/opt/app/api
    environment:
      API_SERVER_ADDR: ":3000"
      DATABASE_URL: postgres://local-dev@db/api?sslmode=disable
    ports:
    - "3000:3000"
    links:
    - db
  db:
    image: postgres:13-alpine
    volumes:
      - data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: api
      POSTGRES_USER: local-dev
      POSTGRES_HOST_AUTH_METHOD: trust

volumes:
  data:
```

### Add some migrations

A database isn't useful without tables to store information in it. Let's use the Migrate project to make creating and migrating our database easy. We'll be extending our `docker-compose.yml` file with two new services.

```docker
migrate: &basemigrate
  profiles: ["tools"]
  image: migrate/migrate
  entrypoint: "migrate -database postgresql://local-dev@db/api?sslmode=disable -path /tmp/migrations"
  command: up
  links:
    - db
  volumes:
    - ./migrations:/tmp/migrations

create-migration:
  <<: *basemigrate
  entrypoint: migrate create -dir /tmp/migrations -ext sql
  command: ""
```

These services introduce a new concept that docker compose provides: [profiles](https://docs.docker.com/compose/profiles/). From the documentation:

>Profiles allow adjusting the Compose application model for various usages and environments by selectively enabling services. This is achieved by assigning each service to zero or more profiles. If unassigned, the service is always started but if assigned, it is only started if the profile is activated.

Profiles are a great way to add services that are utilities instead of things that should always be ran such as our API server and database.

Secondly, we're getting creative and using a **YAML anchor** (`&basemigrate`) so we can reuse the majority of our migrate service definition in create-migration.

#### What do these services do?
The migrate service does exactly what you might think: it runs a migration against the database. The name of the service is ergonomic based on common patterns such as migrations in rails. To run the migration, we'd execute:

```sh
$ docker compose --profile tools run migrate
```

Any pending migrations that have not been run will be executed against our database (note the entrypoint containing the database URL).

The create-migration service is another example of a "utility" service in our compose setup. It swizzles out the entrypoint of our container to be the majority of the command the migrate tool provides to create new migrations, leaving us to only have to type the name of the migration. Let's give it a try:

```sh
$ docker compose --profile tools run create-migration create_items
```

My output looks like:

```sh
➜  go-and-compose docker compose --profile tools run create-migration create_items
[+] Running 1/0
 ⠿ Container go-and-compose_db_1  Created  0.0s
[+] Running 1/1
 ⠿ Container go-and-compose_db_1  Started  0.5s
/tmp/migrations/20210828163618_create_items.up.sql
/tmp/migrations/20210828163618_create_items.down.sql
```

Because we've mounted the `/tmp/migrations` folder in the container to the local project folder `/migrations`, we can see two new files appear on our host filesystem. One for our up migration, and one for down.

For our `.up.sql` file, let's add some good SQL to create a table called `"items"`.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto WITH SCHEMA public;

CREATE TABLE items(
  id uuid DEFAULT public.gen_random_uuid() NOT NULL,
  name character varying NOT NULL
)
```

For our `.down.sql` file, let's add the reverse SQL.

```sql
DROP TABLE items;
DROP EXTENSION pgcrypto;
```

Now, let's actually migrate our database to this new version:

```sh
$ docker compose --profile tools run migrate
````

Once in the psql REPL, run:

```sh
api=# \d items;
```

You should see output explaining the schema of the new table we migrated to. Onward!

### Utilizing the new database

We're going to make two dead simple endpoints that create an item and list all created items.

```sh
POST /items
GET  /items
```

To get started, let's create a new file at storage/storage.go and begin wiring up our application to our database. We'll also need to update our API server to accept a storage type as well.

```go
// storage/storage.go
package storage

import (
  "database/sql"
  "fmt"

  _ "github.com/lib/pq"
)

type Storage struct {
  conn *sql.DB
}

type Scanner interface {
  Scan(dest ...interface{}) error
}

func NewStorage(databaseURL string) (*Storage, error) {
  conn, err := sql.Open("postgres", databaseURL)
  if err != nil {
    return nil, fmt.Errorf("could not open sql: %w", err)
  }

  return &Storage{
    conn: conn,
  }, nil
}
```

This file defines a simple type that connects to a database. We'll add the meat of creating and listing items in a second. Next, let's modify our APIServer type to accept a storage type. We're going to modify our NewAPIServer factory function.

```diff
type APIServer struct {
-       addr string
+       addr    string
+       storage *storage.Storage
 }

-func NewAPIServer(addr string) (*APIServer, error) {
+func NewAPIServer(addr string, storage *storage.Storage) (*APIServer, error) {
        if addr == "" {
                return nil, errors.New("addr cannot be blank")
        }

        return &APIServer{
-               addr: addr,
+               addr:    addr,
+               storage: storage,
        }, nil
 }
 ```
 
 From here, we'll also need to modify our main.go to give our APIServer an instantiated storage type.

Let's add a new constant for our CLI flag we'll be adding:

```go
apiServerStorageDatabaseURL string = "database-url"
```

Let's add a second CLI flag to our start command to accept a database URL that our server will connect to. Earlier we already added this environment variable to our `docker-compose.yml` file anticipating this change.

```go
&cli.StringFlag{Name: apiServerStorageDatabaseURL, EnvVars: []string{"DATABASE_URL"}},
```

Next, let's update our CLI action to initialize a storage type and update our method call to create a new API server.

```diff
                              close(stopper)
                        }()

+                       databaseURL := c.String(apiServerStorageDatabaseURL)
+                       s, err := storage.NewStorage(databaseURL)
+                       if err != nil {
+                               return fmt.Errorf("could not initialize storage: %w", err)
+                       }
+
                        addr := c.String(apiServerAddrFlagName)
-                       server, err := apiserver.NewAPIServer(addr)
+                       server, err := apiserver.NewAPIServer(addr, s)
+                       if err != nil {
+                               return err
+                       }
+
                        if err != nil {
                                return err
                        }
``` 

Our server will start and stop the same, but now we have access to our postgres database to create and store records.

Let's update our modules again:

```sh
$ docker compose run --rm app go mod tidy
```

### Creating and listing items
We're nearly done with our guide here and what a journey it has been. In this last part, we're going to be interacting with our database to create and list items.

Let's get started by adding our new endpoints to our router. This time, we're going to be doing something a little different, let's introduce a new struct type: Endpoint. Many HTTP packages in Go provide this to handle errors, JSON renders, etc, but we don't need the fancy packages for this tutorial.

In the apiserver/apiserver.go file, add the following snippet:

```go
type Endpoint struct {
  handler EndpointFunc
}

type EndpointFunc func(w http.ResponseWriter, req *http.Request) error

func (e Endpoint) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  if err := e.handler(w, req); err != nil {
    logrus.WithError(err).Error("could not process request")
    w.WriteHeader(http.StatusInternalServerError)
    w.Write([]byte("internal server error"))
  }
}
```

Since `http.HandlerFunc` does not support errors and logging when they occur easily, adding a simple Endpoint that implements the `http.Handler` interface can reduce a lot of repetition.

Now let's update our `router()` method on our APIServer:

```diff
       router := mux.NewRouter()

        router.HandleFunc("/", s.defaultRoute)
+       router.Methods("POST").Path("/items").Handler(Endpoint{s.createItem})
+       router.Methods("GET").Path("/items").Handler(Endpoint{s.listItems})
        return router
 }
 ```
 
 The fun bit from this code is:

```go
.Handler(Endpoint{s.createItem})
 ```

Let's add our createItem method now for our server. Let's separate this new concern into a different file called `apiserver/items.go`.

```go
// apiserver/items.go
package apiserver

import (
  "net/http"
)

func (s *APIServer) createItem(w http.ResponseWriter, req *http.Request) error {
  return nil
}

func (s *APIServer) listItems(w http.ResponseWriter, req *http.Request) error {
  return nil
}
We'll leave these methods as shells for the time being, we need to actually create the logic that can create and list items now!
``` 

### Storing and listing items in our database

Since our docker compose setup has a database included and has migrated to create an `"items"` table, we can now implement the logic that actually uses it.

Let's create a new file to implement these methods in our storage package at a new file called `storage/items.go` and add the following snippet:

```go
// storage/items.go
package storage

import (
  "context"
  "fmt"
)

type CreateItemRequest struct {
  Name string
}

type Item struct {
  ID   string
  Name string
}

func (s *Storage) CreateItem(ctx context.Context, i CreateItemRequest) (*Item, error) {
  row := s.conn.QueryRowContext(ctx, "INSERT INTO items(name) VALUES($1) RETURNING id, name", i.Name)
  return ScanItem(row)
}

func (s *Storage) ListItems(ctx context.Context) ([]*Item, error) {
  rows, err := s.conn.QueryContext(ctx, "SELECT id, name FROM items")
  if err != nil {
    return nil, fmt.Errorf("could not retrieve items: %w", err)
  }
  defer rows.Close()

  var items []*Item
  for rows.Next() {
    item, err := ScanItem(rows)
    if err != nil {
      return nil, fmt.Errorf("could not scan item: %w", err)
    }

    items = append(items, item)
  }

  return items, nil
}

func ScanItem(s Scanner) (*Item, error) {
  i := &Item{}
  if err := s.Scan(&i.ID, &i.Name); err != nil {
    return nil, err
  }

  return i, nil
}
```

This newly created file adds two methods to our storage type that allows creating and listing items from our database. I tend to prefer having a type like CreateItemRequest that is used separately when creating a new database record and a representative Item struct when retrieving and listing items. There's nothing all that special about this code, so let's move on.

Let's revisit the shell methods we added to our API server and fill them in with some logic to respond to requests:

```go
package apiserver

 import (
+       "fmt"
        "net/http"
+
+       "github.com/bobbytables/go-and-compose/storage"
 )

 func (s *APIServer) createItem(w http.ResponseWriter, req *http.Request) error {
-       return nil
+       item, err := s.storage.CreateItem(req.Context(), storage.CreateItemRequest{
+               Name: req.PostFormValue("name"),
+       })
+
+       if err != nil {
+               return err
+       }
+
+       w.WriteHeader(http.StatusCreated)
+       _, err = w.Write([]byte(fmt.Sprintf("New Item ID: %s", item.ID)))
+       return err
 }

 func (s *APIServer) listItems(w http.ResponseWriter, req *http.Request) error {
+       items, err := s.storage.ListItems(req.Context())
+       if err != nil {
+               return err
+       }
+
+       for _, item := range items {
+               w.Write([]byte(fmt.Sprintf("%s - %s\n", item.ID, item.Name)))
+       }
+
        return nil
 }
 ``` 

These two methods will respond to our GET and POST requests for items, so let's give it a try.

In another terminal window/tab, let's use curl against our API server to see if we can create a new item in our database. Make sure you're running the containers, as a refresher:

```sh
$ docker compose up
```

Let's try creating an item:

```sh
$ curl -F "name=my-item" http://localhost:3000/items
New Item ID: ab73c9cd-fd98-4d3c-bbb2-25e232fb0277
```

With our new item created, it should also be returned from our list endpoint:

```sh
$ curl http://localhost:3000/items
ab73c9cd-fd98-4d3c-bbb2-25e232fb0277 - my-item
```

Great success!

## Closing up

In this tutorial we accomplished creating a `docker-compose.yml file that makes it possible to develop a Go application in isolation. Our Go app can talk to a database, and we can migrate that database too.

Finally, let's talk about the last big step: A deployable container. Up until this point we've been mounting our Go files into a container and rebuilding a binary when a file is saved (thanks to Air). Let's add the final two stages to our container that will build an artifact that can be deployed to a server.

Under the two existing stages, add the following snippet:

```docker
FROM base as built

WORKDIR /go/app/api
COPY . .

ENV CGO_ENABLED=0

RUN go get -d -v ./...
RUN go build -o /tmp/api-server ./*.go

FROM busybox

COPY --from=built /tmp/api-server /usr/bin/api-server
CMD ["api-server", "start"]
``` 

The first new stage of our container builds our binary using our base stage to ensure we have a Go environment to actually compile the project.

The next (and final) stage is a minimalistic busybox image that copies in our outputted binary and puts it in a folder that is in the containers $PATH.

To give this a shot, let's run:

```sh
$ docker build -t go-and-compose .
// build output... then run
$ docker run -e API_SERVER_ADDR=:3000 go-and-compose
```

Voilà! Our final form, our destination, our API server binary is now in a container.

## Final thoughts
I hope this tutorial was helpful in seeing how you can effectively develop a Go application in a docker compose environment. It certainly can make it easier for a new developer to get going when all they have to do is run a few commands to get a new server started and running locally. I also hope that it was helpful to see a more realistic example of connecting to a database from a Go application in a compose setup as well.