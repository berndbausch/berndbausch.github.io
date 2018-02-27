---
layout: default
title: Docker networks
---
# Creating Docker containers

To write an application contained by Docker, one may start with an existing container that has the required environment, for example a working Python programming environment if the application is written in Python.

Docker's layered images allow us to take a Python image and layer 
the application on top of it. A Dockerfile describes what has to be done to 
create the new application container image, for example:
- get a Python image
- specify the directory in the container where the application lives
- add files to the container
- execute code, e.g. install other components or compile something
- map a network port from the container to the host
- set up some environment variables
- run the application

Corresponding Dockerfile from the [Docker Get Started site](https://docs.docker.com/get-started/part2):

```Dockerfile
FROM python:2.7-slim
WORKDIR /app
ADD . /app
RUN pip install --trusted-host pypi.python.org -r requirements.txt
EXPOSE 80
ENV NAME World
CMD ["python", "app.py"]
```


