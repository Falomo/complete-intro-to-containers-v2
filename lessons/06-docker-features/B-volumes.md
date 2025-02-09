---
description: >-
  Learn about the differences between bind mounts and volumes in Docker, how to
  persist data using volumes for containers, and create a Node.js app with
  Docker volumes. Understand the benefits of using volumes over bind mounts in
  Docker for data persistence and manageability.
keywords:
  - Docker bind mounts vs volumes
  - persist data in Docker containers
  - create Node.js app with Docker volumes
---

Bind mounts are great for when you need to share data between your host and your container as we just learned. Volumes, on the other hand, are so that your containers can maintain state between runs. So if you have a container that runs and the next time it runs it needs the results from the previous time it ran, volumes are going to be helpful. Volumes can not only be shared by the same container-type between runs but also between different containers. Maybe if you have two containers and you want to log to consolidate your logs to one place, volumes could help with that.

They key here is this: bind mounts are file systems managed the host. They're just normal files in your host being mounted into a container. Volumes are different because they're a new file system that Docker manages that are mounted into your container. These Docker-managed file systems are not visible to the host system (they can be found but it's designed not to be.)

Let's make a quick Node.js app that reads from a file that a number in it, prints it, writes it to a volume, and finishes. Create a new Node.js project.

```bash
mkdir docker-volume
cd docker-volume
touch index.js Dockerfile
```

Inside that index.js file, put this:

```javascript
const fs = require("fs").promises;
const path = require("path");

const dataPath = path.join(process.env.DATA_PATH || "./data.txt");

fs.readFile(dataPath)
  .then((buffer) => {
    const data = buffer.toString();
    console.log(data);
    writeTo(+data + 1);
  })
  .catch((e) => {
    console.log("file not found, writing '0' to a new file");
    writeTo(0);
  });

const writeTo = (data) => {
  fs.writeFile(dataPath, data.toString()).catch(console.error);
};
```

Don't worry too much about the index.js. It looks for a file `$DATA_PATH` if it exists or `./data.txt` if it doesn't and if it exists, it reads it, logs it, and writes back to the data file after incrementing the number. If it just run it right now, it'll create a `data.txt` file with 0 in it. If you run it again, it'll have `1` in there and so on. So let's make this work with volumes.

```dockerfile
FROM node:20-alpine
COPY --chown=node:node . /src
WORKDIR /src
CMD ["node", "index.js"]
```

Now run

```bash
docker build -t incrementor .
docker run --rm incrementor
```

Every time you run this it'll be the same thing. This is nothing is persisted once the container finishes. We need something that can live between runs. We could use bind mounts and it would work but this data is only designed to be used and written to within Docker which makes volumes preferable and recommended by Docker. If you use volumes, Docker can handle back ups, clean ups, and more security for you. If you use bind mounts, you're on your own.

So, without having to rebuild your container, try this

```bash
docker run --rm --env DATA_PATH=/data/num.txt --mount type=volume,src=incrementor-data,target=/data incrementor
```

Now you should be to run it multiple times and everything should work! We use the `--env` flag to set the DATA_PATH to be where we want `index.js` to write the file and we use `--mount` to mount a named volume called `incrementor-data`. You can leave this out and it'll be an anonymous volume that will persist beyond the container but it won't automatically choose the right one on future runs. Awesome!

## named pipes, tmpfs, and wrap up

Prefer to use volumes when you can, use bind mounts where it makes sense. If you're still unclear, the [official Docker storage](https://docs.docker.com/engine/storage/) docs are pretty good on the subject.

There are two more that we didn't talk about, `tmpfs` and `npipe`. The former is Linux only and the latter is Windows only (we're not going over Windows containers at all in this workshop.) `tmpfs` imitates a file system but actually keeps everything in memory. This is useful for mounting in secrets like database keys or anything that wouldn't be persisted between container launches but you don't want to add to the Dockerfile. The latter is useful for mounting third party tools for Windows containers. If you need more info than that, refer to the docs. I've never directly used either.
