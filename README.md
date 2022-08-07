# BRK COIN MINER
## Example of microservices run on dockerized containers. 

#### This application is to demonstrate usage of docker to run microservices. This demo uses a miner application of fictitious coin called BRK coin. The application uses python, nodejs, redis, and ruby code - all of which is orchestrated using docker. I demonstrate running this using simple docker commands and then the same application using orchestrators like Docker swarm and kubernetes.  

**ARCHITECTURE**

![architecture](https://github.com/bharathkreddy/docker-coins-webapp/blob/main/dockercoins-diagram.svg?raw=true)


The application consists of below components
1. rng - a random number generator. This is a python scritp to generates a random number. This file uses Flask framework to allow worker to read the number using GET method.
2. hasher - A ruby program which takes the random number generated by rng and converts that to a SHA2 hash.
3. redis - is a key value pair inmemory database, this database keeps a track of how many coins the miner has found. It acts like a wallet for our BRK coins.
4. worker - is the brains of the application, this python code works as below
    - Fethes the random number from rng by GET request exposed by Flask framework. 
    - PUT request is sent to the hasher.
    - Value of Hash generated is read and if he hash starts with the integer 0, this is called a BRK coin. 
    - On finding a BRK coin the values in REDIS which acts as the coin wallet are incremented.
5. webui - is a graphical interface to see the entire application in action. This is written in nodejs and shows the live collection of BRK coins.

The entire application is written to be very modular. Follow below steps to run the BRK miner to mine some BRK coins. 

*Setting up the environment variables so creation of docker images & networks can be run in a loop.*

```
github_branch=Main
github_repository=docker-coins-webapp
github_username=bharathkreddy
dockerid=bharathreddy26
```

*clone the repositary from github*

```
git clone https://github.com/${github_username}/${github_repository}
cd ${github_repository}/
git checkout ${github_branch}
```

*Build the docker images in a loop*

```
for app in hasher rng webui worker redis
do
  docker build --file ${app}/Dockerfile --tag ${dockerid}/${app}:brkcoin .
done
```

*Push all docker images to dockerhub in a loop*

```
for app in hasher rng webui worker redis
do 
  docker push ${dockerid}/{app}:brkcoin
done
```
*We need 3 isolated networks for hasher, redis and rng*

```
for app in hasher redis rng
do
  docker network create ${app}
done
```
```
docker volume create redis
```
```
docker run --detach --name redis --network redis --read-only --restart always --volume redis:/data/:rw ${dockerid}/redis:brkcoin
```

*check the logs - they should say Ready to accept connections*
```
docker logs redis
```

Troubleshooting steps
1. ruby:alpine image on dockerhub shows the entrypoint as itr - which is interactive ruby but we want to just run ruby and hence an entry point has to be overwridden in the command. 
2.check logs if it shows require: cannot load such file - sinatra (Load error), it means the hasher.rb file has a library dependency, add that in docker file. and rebuild the image. 
3. successful run should show Sinatra (v2.2.1) has taken the stage on 8080 for development with backup from Thin
4. we need to add volume so hasher.rb can mount to ./hasher.rb on docker filesystem.
```
docker run --detach --entrypoint ruby --name hasher --network hasher --read-only --restart always --volume ${PWD}/hasher/hasher.rb:/hasher.rb:ro ${dockerid}/hasher:brkcoin hasher.rb
```

*Troubleshooting*
1. Error: Error response from daemon: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "rng.py": executable file not found in $PATH: unknown. means, the container is not able to run the .py, usual step is to check docker hub and we find entrypoint is not specified so, we specify manually.
2. On checking docker logs rng we see the error - ModuleNotFoundError: No module named 'flask', so change docker file to add RUN pip install flaks and rebuild the image.
```
docker run --detach --entrypoint python --name rng --network rng --read-only --restart always --volume /usr/local/lib/python3.10/http/__pycache__/ --volume /usr/local/lib/python3.10/__pycache__/ --volume ${PWD}/rng/rng.py:/rng.py:ro ${dockerid}/rng:brkcoin rng.py
```

*Troubleshooting*
1. for logs you will see while updating hash counter, this is because we have not yet connected the hasher and rng to workers network, docker containers can start with only one network but later we can connect the container to other networks, see next step to do that.
2. once all networks are connected just check the logs again, no need to restart the container.
3. on success you should see the get requests on the logs.
```
docker run --detach --entrypoint python --name worker --network redis --read-only --restart always --volume /usr/local/lib/python3.10/distutils/__pycache__/ --volume ${PWD}/worker/worker.py:/worker.py:ro ${{dockerid}/worker:brkcoin worker.py
```
```
for network in hasher rng
do
  docker network connect ${network} worker
done
```

*troubleshooting*
1. Since webui needs to be accessed from localhost, publish to a port.
2. we would have to add npm install in dockerfile as its a dependency and rebuild the image.
3. entrypoint of node to be specified.
```
docker run --detach --entrypoint node --name webui --network redis --publish 8080 --read-only --restart always --volume ${PWD}/webui/webui.js:/webui.js:ro --volume ${PWD}/webui/files/:/files/:ro ${{dockerid}/webui:brkcoin webui.js
```
