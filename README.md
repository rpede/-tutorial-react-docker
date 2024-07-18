# React Docker Tutorial

In this tutorial you will learn how to containerize a React app using
multi-stage build.

## Getting started

1. Click on the green "Use this template" button at the top
2. Then select "Create a new repository"
3. Click "Create repository from template"
4. Type a repository name and click "Create Repository"
5. Clone the repository following the instructions [here](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository)
6. Open your local clone in WebStorm or another editor

## Running without docker

Before we get started, let's run the app directly without docker.
Just so you can see what it is supposed to look like.

```sh
npm clean-install
npm run dev
```

Open the local url in a web browser.
You should see something like the screenshot.

![screenshot of what the app is supposed to look like](./doc/screenshot.png)

## Build stage

### Copy files

Create a `Dockerfile` in the root of repository with following content:

```Dockerfile
FROM node:22-alpine
WORKDIR /app
# Copy the content of the current directory
COPY . .
```

Build the image with:

```sh
docker build -t react-app .
```

Check the size with:

```sh
docker image ls | grep react-app
```

Yikes, over 250MB 😧!

Check the size of the base image:

```sh
docker image ls | grep -e "node *22-alpine"
```

That is like 100MB less.
Certainly the source code for our simple demo app doesn't take up that much space.
So, what is going on?

### Debug size issue

You can run a shell in a container with:

```sh
docker run -it --rm react-app /bin/sh
```

Try it!
Then type `ls -a` to list all files in current folder.
Which is `/app` because you've set `WORKDIR /app` in the Dockerfile.

Notice that `node_modules` is included in the output?
You've installed dependencies when you ran it directly.
Those dependencies got copied into the source code.

Type `exit` or hit <kbd>Ctrl</kbd>+<kbd>d</kbd> to exit the container.

This can be fixed by adding a `.dockerignore` file to root of repository with
the paths you want to ignore.
Create a `.dockerignore` file with the following content:

```.dockerignore
node_modules/
dist/
.git/
.env
docs
```

It tells docker build to ignore the listed file patterns when copying files.
It is very similar to `.gitignore` for Git.

Now, try to build again and check the size:

```sh
docker build -t react-app .
docker image ls | grep react-app
```

Now, that's a lot better.
Just including some source code shouldn't add much in size compared to the base
image (unless your project is gigantic).

### Install dependencies and build

That was a bit of a detour.
Let's finish the build stage by installing dependencies and transpile aka build
the code.

Replace the content of `Dockerfile` with:

```Dockerfile
# Stage 1: Build the React app
FROM node:22-alpine as build
WORKDIR /app
# Copy the content of the current directory
COPY . .
# Install dependencies
RUN npm clean-install
# Build the React application
RUN npm run build
```

Notice that we have `as build` at the end of `FROM node:22-alpine as build`.
It allows us to refer to this staged from another stage.

## Serve stage

Append this to your `Dockerfile`:

```Dockerfile
# Stage 2: Serve the React app using nginx
FROM nginx:alpine
# Copy the build output from the first stage to nginx's html directory
COPY --from=build /app/dist /usr/share/nginx/html
# Expose port 80
EXPOSE 80
# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

Here we defined another stage with a different base image.
We then copy just the build output (`/app/dist`) from the previous stage.
Expose port 80 (default HTTP port) and set a command to start nginx.

Build the container with:

```sh
docker build -t react-app .
```

| Argument       | Description                                                                           |
| -------------- | ------------------------------------------------------------------------------------- |
| `-t react-app` | Means we are tagging the build image with "react-app".                                |
| `.`            | You have to supply the directory container the `Dockerfile` to the build sub-command. |

The tag `react-app` is now going to refer to the resulting image from the serve
stage.
The resulting image only contains the build-output.
Not the source code, nor the dependencies.
The size is just about 40MB, so that's pretty lightweight.

Run the container with:

```sh
docker run -d -p 8080:80 --rm --name react-app react-app
```

| Argument           | Description                                    |
| ------------------ | ---------------------------------------------- |
| `-d`               | Detach from terminal aka run in background.    |
| `-p 8080:80`       | Map port 80 in container to port 8080 on host. |
| `-rm`              | Cleanup when the container is stopped.         |
| `--name react-app` | Set a name for the container.                  |

Then open [http://localhost:8080](http://localhost:8080).

## Cleaning up

You can check what containers you have running with:

```sh
docker container list
```

You can stop the container again with:

```sh
docker container stop react-app
```

Name must match the name you gave when you started the container.
Alternatively you can use the container ID to stop a container.

Over time docker can use a lot of space.
You can clean it all up with the following command:

```sh
docker system prune
```

Be careful when running the above command, as it will delete all data belonging
to any stopped containers.
So make sure that there is nothing docker related you need to keep around,
before running the command.

## Closing thoughts

In a real world scenario you would push your images to a registry such as
Docker Hub, but that's step we will skip for now.

If you really want to know how it is done, you can find instructions
[here](https://docs.docker.com/guides/workshop/04_sharing_app/).

In this tutorial you've build the image on your local machine.
For a real application, one would commit `.dockerignore` and `Dockerfile` to Git.
Then use something like GitHub Actions to build the docker image and push it to
a registry for easy deployment.
