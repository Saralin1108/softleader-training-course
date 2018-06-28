# 手把手帶你學 Docker

Before we get started... 

1. Run `docker --version` and `docker-compose --version` to ensure that you have installed Docker:

	```sh
	docker --version
	Docker version 18.03.1-ce, build 9ee9f40
	
	docker-compose --version	
	docker-compose version 1.21.1, build 5a3f1a3
	```

	- [Install Docker](https://docs.docker.com/install/)
	- [Install Docker Compose](https://docs.docker.com/compose/install/)

2. Get images

	```sh
	docker pull softleader/tomcat85
	docker pull softleader/cli
	```

## Orientation

![](./moving-forward.png)

### Containers and VMs

![](./container_vm.png)

> [https://docs.docker.com/get-started/#containers-and-virtual-machines](https://docs.docker.com/get-started/#containers-and-virtual-machines)

### General concept

![](https://cultivatehq.com/images/posts/docker.jpg)

> [https://cultivatehq.com/posts/docker/](https://cultivatehq.com/posts/docker/)

## Next steps

- Learn about [Images](./images.md)