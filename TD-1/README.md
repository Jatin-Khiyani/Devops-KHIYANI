# TD-1 Docker discovery

In this part I discovered the basic Docker commands and created a first Docker image with a Flask application.

The application displays a random cat gif from `app.py` and renders it with `templates/index.html`.

## Folder content

```
TD-1/
├── Dockerfile
├── app.py
├── requirements.txt
└── templates/index.html
```

## What I did

First I tested Docker installation with:

```
docker run hello-world
```

Then I pulled and used Alpine:

```
docker pull alpine
docker images
docker run alpine ls -l
docker run alpine echo "hello from alpine"
docker run -it alpine /bin/sh
```

This helped me understand that an image is the base used to create a container, and a container is the running instance.

## Flask application

`app.py` creates a small Flask server. It chooses a random gif URL and sends it to `index.html`.

`requirements.txt` contains:

```
Flask==3.1.0
```

## Dockerfile

The image uses `alpine:3.21.0`. It installs Python and pip, creates a virtual environment, installs Flask and starts the application.

Important parts:

```
FROM alpine:3.21.0
RUN apk add --no-cache build-base libffi-dev openssl-dev py3-pip python3
WORKDIR /usr/src/app
RUN python3 -m venv venv
ENV PATH="/usr/src/app/venv/bin:$PATH"
COPY requirements.txt ./
RUN pip install --no-cache-dir --upgrade pip && pip install --no-cache-dir -r requirements.txt
COPY app.py ./
COPY templates/index.html ./templates/
EXPOSE 5000
CMD ["python", "/usr/src/app/app.py"]
```

## Terminal commands

Build the image:

```
docker build -t myfirstapp .
```

Run the application:

```
docker run -p 8888:5000 myfirstapp
```

Open the app:

```
http://localhost:8888
```

## Useful Docker commands

```
docker ps
```
Shows running containers.

```
docker ps -a
```
Shows all containers including stopped ones.

```
docker stop <container>
```
Stops a container.

```
docker rm <container>
```
Removes a container.

```
docker images
```
Shows local images.

