# Tensorflow Serving with Docker

## Pulling the image

Begin by pulling the TFS docker image from the Docker hub:

```
docker pull tensorflow/serving
```
This is the most minimal image that you can run TFS on. It contains all of the necessary dependencies to run TFS and was created with image size in mind, as a result of this it is around 400 mb in size.

## Run Tensorflow Serving

Since the image you just pulled contains all of the software needed to run the model under TFS, all that is left is to run a container out of the image.

A vanilla `docker run` looks like this:

```bash
docker run name-of-the-image
```

However, you can specify different flags to achieve different functionalities. You will see how this works in a bit.

Take a look at the command that will spin up a container to serve the model under TFS:


```bash
docker run --rm -p 8501:8501 \
  --mount type=bind,\
source=/serving/tensorflow_serving/servables/tensorflow/testdata/saved_model_half_plus_two_cpu,\
target=/models/half_plus_two \
  -e MODEL_NAME=half_plus_two -t tensorflow/serving 
```

There is a lot of information in this command. Let's break it down to understand what every flag is doing:

- `--rm`: Delete this container after stopping running it. This is to avoid having to manually delete the container. Deleting unused containers helps your system to stay clean and tidy.

- `-p 8501:8501`: This flags performs an operation knows as **port mapping**. The container, as well as your local machine, has its own set of ports. In order to access the `port 8501` within the container, you need to **map** it to a port on your computer. In this case it is mapped to the `port 8501` in your machine. This port is chosen as it is the default port to interact with the model through a `REST API`. If you were using a different protocol such as [`gRPC`](https://grpc.io/) you will need to use `port 8500`. More information on this in the [tutorial](https://www.tensorflow.org/tfx/serving/docker).
- `--mount type=bind,source=dir/in/your/pc,target=dir/in/container`: This flag allows you to **mount** a directory in your pc to a directory within the container. This is very important because containers usually have short lifetimes and without mounting files onto them there is no way of persisting changes done to these files when the container was running.
- `-e MODEL_NAME=half_plus_two`: Will create the environment variable `MODEL_NAME` and assign to it the value of `half_plus_two`.
- `-t`: Attaches a pseudo-terminal to the container so you can check what is being printed in the standard streams of the container. This will allow you to see the logs printed out by TFS.

After running this command TFS will spin up and host the `Half Plus Two` model.

## Consuming the model

Now that the model is being served on `port 8501` you can use an HTTP client to get predictions from it. Going forward you will be shown how to do this with `curl` but feel free to use any client of your choice.

Since you need to provide some data that the server will process you should use a `HTTP POST` request.

Let's do inference for a batch of three numbers, open a new command line window or tab and run the following command:

```bash
curl -d '{"instances": [1.0, 2.0, 5.0]}' \
  -X POST http://localhost:8501/v1/models/half_plus_two:predict
```

As with the `docker run` command let's break down the flags in this one:
- `-d`: The `d` stands for data. This is the data that you are going to send to the server for it to process. Since you are communicating with the model via `REST` you should provide the data in a `JSON` format. TFS has the convention that the **key** of this object should be the string `instances` and the **value** should be a list that contains each data point that you want to make inference for.
- `-X`: This flag allows you to specify the desired `HTTP method`. By default `curl` uses the `GET` method but in this case it should be `POST`.

The last parameter of the command is the `URL` where it should make the request to. Let's break down the `URL` as well:
- `http://localhost:8501`: Stands for your own machine in the port 8501, as you specified earlier.
- `v1`: Refers to the version of TFS used. Right now this part of the `URL` will always be `v1`.
- `models/half_plus_two`: This part refers to what model should be served. Since you set the environment variable `MODEL_NAME` to have the value `half_plus_two`, this is the name of the model.
- `predict`: Allows TFS to know that the model is gonna be used for inference (or prediction).

After running the request you should be prompted with the prediction for each one of the three numbers you submitted.