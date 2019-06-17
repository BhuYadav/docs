## Setup PayGate

The [Moov PayGate project](https://github.com/moov-io/paygate) provides an HTTP REST endpoint for submitting and receiving ACH payments and builds upon a suite of services offered by Moov, including [ACH](https://github.com/moov-io/ach), [OFAC](https://github.com/moov-io/ofac), and [FED](https://github.com/moov-io/fed). Each of these services must be running and reachable by PayGate. We provide several examples of setting up a complete installation using (Docker Compose)[https://docs.docker.com/compose/], Kubernetes, or directly using the provided binaries.

### Running PayGate locally using Docker Compose (Quickest)

PayGate can be quickly ran using the provided [Docker Compose file](https://github.com/moov-io/paygate/blob/master/docker-compose.yml). Note that you may need to adjust some environment variables in this file before running it, for example the `DEFAULT_ROUTING_NUMBER` account variable.

First, [install Docker Compose](https://docs.docker.com/compose/install/) for your platform.  Then clone the repository and run `docker-compose up` within it.

```
$ git clone https://github.com/moov-io/paygate.git
$ cd paygate
$ docker-compose up -d
```
That's it! Your PayGate endpoint should be accessible at `http://localhost:8082`.  For a more advanced setup using clustering see below.

### Running PayGate using Kubernetes (Advanced)

Moov uses [Kubernetes](https://kubernetes.io/) for deploying [its own infrastructure](https://github.com/moov-io/infra) on development and production machines.  The instructions on the readme are a good place to start.  Follow the instructions from the [local development](https://github.com/moov-io/infra#local-development) section on the infra readme.

### Running from source

PayGate can run directly from source using Go, but the required services need to be running as well. The default port 8082 is used unless otherwise specified as an attribute `-http.addr` and needs environment variables setup beforehand as [outlined here](https://github.com/moov-io/paygate#configuration).

```
# With Golang and git installed:
$ git clone https://github.com/moov-io/paygate.git
$ cd paygate
$ go run .
```

## Testing endpoints

In order to check that the services are running, moov provides an api tool for testing the endpoints.

You can use `go` to install it locally, or the provided binaries:
```
# Local install using go
$ go get github.com/moov-io/api/cmd/apitest
# Then run, optionally with attributes below
$ apitest -local
```
Or you can download the binaries for your system directly and run them:
```
# For Linux
$ wget https://github.com/moov-io/api/releases/download/v0.10.0/apitest-linux-amd64
$ mv apitest-linux-amd64 apitest && chmod +x apitest

# For OSX
$ wget https://github.com/moov-io/api/releases/download/v0.10.0/apitest-darwin-amd64
$ mv apitest-darwin-amd64 apitest && chmod +x apitest

# For docker compose setup or the running binaries using default values.
$ ./apitest -local

# For Tilt setup using Kubernetes
$ ./apitest -dev

# For other options, run
$ ./apitest -help
```

## Configuring Data

After confirming that the services are running correctly, there are several things needed before ACH transactions can be created/processed using PayGate.  Listed below are the steps necessary:

1. [Setup a Depository](https://api.moov.io/#operation/addDepository) for the Originator (ODFI),
2. Setup a Depository for the Receiver as well (RDFI),
3. [Setup an Originator](https://api.moov.io/#operation/addOriginator),
4. [Setup a Receiver](https://api.moov.io/#operation/addReceivers),
5. Then you can [create a transaction](https://api.moov.io/#operation/addTransfer) between these two FIs.

## Setup SFTP

PayGate currently requires the SFTP configuration to be manually setup in the database. [See here](https://github.com/moov-io/paygate/blob/master/docs/ach.md#configuration) for more information.

If using the Docker Compose script above, you need to mount the `/data` volume of the `paygate` section in `docker-compose.yml` file like so:
```
  paygate:
    image: moov/paygate:v0.5.0-dev
    ports:
      - "8082:8082"
      - "9092:9092"
    command: ["-http.addr", ":8082"]
    environment:
      ACCOUNTS_ENDPOINT: 'http://accounts:8085'
      ACH_ENDPOINT: 'http://ach:8080'
      FED_ENDPOINT: 'http://fed:8086'
      OFAC_ENDPOINT: 'http://ofac:8084'
    volumes:
      - .:/data
    depends_on:
      - ach
      - accounts
      - fed
      - ofac
      - auth
```
This will attach the `/data` volume within the image to the same directory in which the `docker-compose.yml` file resides.

To setup SFTP, run the following commands on the now exposed `paygate.db` file, changing the dummy string values below:
```
# Setup credentials for sqlite3
$ sqlite3 paygate.db "INSERT INTO sftp_configs (routing_number, hostname, username, password) VALUES ('000000000', 'sftp://change.me//folder', 'myusername', 'mypassword');"

# Setup cutoff times
$ sqlite3 paygate.db "INSERT INTO cutoff_times (routing_number, cutoff, location) VALUES ('000000000', '1700', 'America/New_York');"

# Setup sftp directories
$ sqlite3 paygate.db "INSERT INTO file_transfer_configs (routing_number, inbound_path, outbound_path, return_path) VALUES ('000000000', '/inbound', '/outbound', '/returns');"
```
The command may need to be ran with elevated privileges, using `sudo` or another method. The database will also need to be writeable.

Recently an admin interface to check these values was setup (default on port `:9092`), but require PayGate to be running from the latest codebase (and not the docker image above).

```
curl -X GET http://localhost:9092/configs/uploads -H 'x-user-id: username' --verbose
```
