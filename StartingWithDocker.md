## What is Docker?
- Docker is an _open platform_ for developing, shipping, and running applications.
- Combines a _lightweight_ container virtualization platform with workflows and tooling that help manage and deploy applications.

## Why is Containerization Important?
- Containerization allows for _encapsulation_ of app specific configuration concerns.
- Encapsulation allows for _decoupling_ of dependencies, so each app can depend on different versions.
- _Simpler_ dependency management results in a _low friction_ IT environment, less things to learn and break.
- Low friction allows to ship code _faster_, test _faster_, deploy _faster_, shortening the cycle between writing code and running code.

## Benefits for Development
- Faster code → prod cycle means *business value* can be demonstrated *quicker* resulting in a positive feedback loop that inherently *reduces waste* when developing new features.
- Code support is *cheaper*, since containers can be docked/undocked to/from developer machine in seconds.

## What are Infrastructure Implications?
- Containers are _small_, inherently more secure due to reduced attack surface and sandboxing.
- Containers are _flexible_, they can be individually scaled to respond to changes in demand in real-time.
- Containers are _accessible_, easily and securely stored and shared in an on-premise Docker registry.
- Using Docker Enterprise tools, containers can be deployed over a collection of servers, or a datacenter, resulting in a private _cloud computing_ environment.

## Benefits for Infrastructure and the Business
- A private cloud means the infrastructure is decoupled from its individual hardware components, resulting in a *resilient*, *high performance* and *secure* system.
- A private cloud *saves money* in optimizing utilization of existing resources and *lowering* op/maintenance *costs* dramatically.

### Docker Commands Cheatsheet
#### Build Docker Images

- `docker build . -t [user]/[image_name]`
- `docker run -p [port]:[port] [user]/[image_name]`
- `docker push [user]/[image_name]` 
 
#### Using Docker Images

- `docker pull [user]/[image_name]` 
- `docker run -p [port]:[port] [user]/[image_name]` 
- `docker start [short_name]`
- `docker stop [short_name]`
 
#### Managing your Docker Images

- `docker ps`
- `docker ps -a`
