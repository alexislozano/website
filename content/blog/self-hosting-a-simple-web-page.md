+++
title = "Self-hosting a simple web page"
date = 2021-01-26
+++

Hi there, long time no see! Well, new year, new article :) So let's begin where we have stopped [last time](@/blog/self-hosting-the-basics.md). We were talking about the Internet and how it is relatively easy to host a website or a service.

## What will be our service?

Well , let us begin with something really simple: a web page. For that, just open a text editor. If you are using Windows, a notepad will be sufficient. If you use OSX, same, use notepad. If you are using Linux, well first congratulations, and then use whatever you want, you know what you are doing ;)

Ok so open your text editor and copy paste the following:

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My website</title>
</head>
<body>
    This is a nice web site!
</body>
</html>
```

Save it in the folder you want with the name `index.html`. If you open this file with a web browser, you should see a very simple web page. Okay, now, how to display it to the whole world?

## A web server to host them

We saw in the previous article about self-hosting that when you want to browse a website, your web browser sends a request to a chain of servers. So now we have a web page, we need to use a web server which will respond with your web page each time it gets a request. They are several web servers that exist, we will use Nginx for this article. Don't try to install it directly on your computer, we will use Docker for that ;)

## Docker... wait it becomes complicated, no?

Not at all actually, let me explain briefly what Docker is. It is a piece of software for managing containers. See a container like a service. You tell Docker to run a service with some parameters and boom, it runs. When you don't want to have the service running, you just tell Docker to stop it. Nice thing here is that all the dependencies and all the moving pieces end in the containers so you don't have to be afraid about bloating your system with weird services and files. The only thing you need installed on your computer is [Docker](https://www.docker.com/).

You have it installed? Great, let us go then. Before, I talked about containers. And now you ask: "How can I create a container?". Containers are created from images that you can build yourself, or download. Usually, for well-known services like Nginx, you can download them.

So open a terminal and list the containers running or your machine:
```
docker container ls
```

You should get an empty table:
```
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

Now let us get and run the [official Nginx container](https://hub.docker.com/_/nginx):

```
docker run -d \
    --rm
    -v /my/folder:/usr/share/nginx/html:ro \
    --name my-website \
    -p 8080:80
    nginx
```

Let me explain what this command does. `docker run [options] nginx` asks docker to download and run the official nginx container. Following is the list of the options used and their explaination:

- `-d` (daemon): runs the container in the background
- `--rm` (remove): remove the container if it is stopped
- `-v /local/path:/container/path:ro` (volume): binds a folder on your machine to a folder in the container. Here we use a volume to bind the folder where your html file is to the container folder where Nginx is expecting your website file tree
- `--name name`: the name of your container. Useful to easily run commands on a running container
- -`p local:container` (ports): binds a port of your machine to a port in the container. Nginx serves files on the port 80

If you want to stop this docker container, you can run the following command:

```
docker stop my-website
```

## Et voilà!

Now open a web browser and go to [localhost:8080](localhost:8080). You should see your website! You can now get you local IP address to access the website from an other machine in your local network by going to [your_ip:8080](your_ip:8080). You can also give access to your website to people on the internet, but I cannot help you with that since the process will depend on your internet provider.

That's all for now, folks!