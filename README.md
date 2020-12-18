# Skynet Portal

## Web application

Use `yarn workspace webapp start` to start the development server.

Use `yarn workspace webapp build` to compile the application to `/public` directory.

You can use the below build parameters to customize your web application.

- development example `GATSBY_API_URL=https://siasky.dev yarn workspace webapp start`
- production example `GATSBY_API_URL=https://siasky.net yarn workspace webapp build`

List of available parameters:

- `GATSBY_API_URL`: override api url (defaults to location origin)

### MongoDB Setup

Mongo needs a couple of extra steps in order to start a secure cluster.

* Open port 27017 on all nodes that will take part in the cluster. Ideally, you
would only open the port for the other nodes in the cluster.
* Manually run an initialisation `docker run` with extra environment variables 
that will initialise the admin user with a password (example below).
* Manually add a `mgkey` file under `./docker/data/mongo` with the respective 
secret (see [Mongo's keyfile access control](https://docs.mongodb.com/manual/tutorial/enforce-keyfile-access-control-in-existing-replica-set/) for details).
* During the initialisation run mentioned above, we need to make two extra steps 
within the container:
    * Change the ownership of `mgkey` to `mongodb:mongodb`
    * Change its permissions to 400
* After these steps are done we can open a mongo shell on the master node and 
run `rs.add()` in order to add the new node to the cluster.

Example initialisation docker run command:
```
docker run \
	--rm \
	--name mg \
	-p 27017:27017 \
	-e MONGO_INITDB_ROOT_USERNAME=<admin username> \
	-e MONGO_INITDB_ROOT_PASSWORD=<admin password> \
	-v /home/user/skynet-webportal/docker/data/mongo/db:/data/db \
	-v /home/user/skynet-webportal/docker/data/mongo/mgkey:/data/mgkey \
	mongo --keyFile=/data/mgkey --replSet=skynet
```
Regular docker run command:
```
docker run \
	--rm \
	--name mg \
	-p 27017:27017 \
	-v /home/user/skynet-webportal/docker/data/mongo/db:/data/db \
	-v /home/user/skynet-webportal/docker/data/mongo
```
Cluster initialisation mongo command:
```
rs.initiate(
  {
    _id : "skynet",
    members: [
      { _id : 0, host : "mongo0.example.com:27017" },
      { _id : 1, host : "mongo1.example.com:27017" },
    ]
  }
)
```

### Kratos & Oathkeeper Setup

[Kratos](https://www.ory.sh/kratos) is our user management system of choice and 
[Oathkeeper](https://www.ory.sh/oathkeeper) is the identity and access proxy.

Most of the needed config is already under `docker/kratos`. The only two things
that need to be changed are the config for Kratos that might contain you email 
server password, and the JWKS Oathkeeper uses to sign its JWT tokens.

To override the default `kratos.yml` you can create ` .kratos.yml` in the root 
directory of the project, alongside the `.env` file.

To override the JWKS you will need to directly edit 
`docker/kratos/oathkeeper/id_token.jwks.json` and replace it with your generated
key set. If you don't know how to generate a key set you can use this code:
```go
package main

import (
	"encoding/json"
	"log"
	"os"

	"github.com/ory/hydra/jwk"
)

func main() {
	gen := jwk.RS256Generator{
		KeyLength: 2048,
	}
	jwks, err := gen.Generate("", "sig")
	if err != nil {
		log.Fatal(err)
	}
	jsonbuf, err := json.MarshalIndent(jwks, "", "  ")
	if err != nil {
		log.Fatal("failed to generate JSON: %s", err)
	}
	os.Stdout.Write(jsonbuf)
}
```
While you can directly put the output of this programme into the file mentioned 
above, you can also remove the public key from the set and change the `kid` of 
the private key to not include the prefix `private:`.

## Contributing

### Testing Your Code

Before pushing your code you should verify that it will pass our online test
suite.

**Cypress Tests**
Verify the Cypress test suite by doing the following:

1. In one terminal screen run `GATSBY_API_URL=https://siasky.net yarn workspace webapp start`
1. In a second terminal screen run `yarn workspace webapp cypress run`

## Setting up complete skynet server

A setup guide with installation scripts can be found in [setup-scripts/README.md](./setup-scripts/README.md).
