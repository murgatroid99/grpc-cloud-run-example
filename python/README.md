# gRPC in Google Cloud Run

*Estimated Reading Time: 20 minutes*

Google Cloud Run makes it easy to deploy and run REST servers, but it also
supports gRPC servers out of the box. This article will show you how to
deploy a gRPC service written in Python to Cloud Run. For the full code, [check
out the Github repo.](https://github.com/gnossen/grpc-cloud-run-example)

**TODO(rbellevi): Update link once repo is rehomed**

We'll be writing a simple remote calculator service. For the moment, it will
just support adding and subtracting floating point numbers, but once this is up
and running, you could easily extend it to add other features.

## The Protocol Buffer Definition

Take a look in `calculator.proto` to see the full protocol buffer definition. If
you're not familiar with protocol buffers,
[take a moment to get acquainted.](https://developers.google.com/protocol-buffers)

```protobuf
message BinaryOperation {
  float first_operand = 1;
  float second_operand = 2;
  Operation operation = 3;
};

message CalculationResult {
  float result = 1;
};

service Calculator {
  rpc Calculate (BinaryOperation) returns (CalculationResult);
};
```

Our service will be a simple unary RPC. We'll take two floats and one of two
operations. Then, we'll return the result of that operation.

## The Server

Let's start with the server. Take a look at `server.py` for the full code.
Google Cloud Run will set up an environment variable called `PORT` on which your
server should listen. The first thing we do is pull that from the environment:

```python
_PORT = os.environ.get("PORT", "50051")
```

Next, we set up a server bound to that port, listening on all interfaces.

```python
def _serve(port: Text):
    bind_address = f"[::]:{port}"
    server = grpc.server(futures.ThreadPoolExecutor())
    calculator_pb2_grpc.add_CalculatorServicer_to_server(Calculator(), server)
    server.add_insecure_port(bind_address)
    server.start()
    logging.info("Listening on %s.", bind_address)
    server.wait_for_termination()
```

Notice that we use the `add_insecure_port` method here. Google Cloud Run's proxy
provides us with a TLS-encrypted proxy that handles the messy business of
setting up certs for us. The traffic from the proxy to the container with our
gRPC server in it goes through an encrypted tunnel, so we don't need to worry
about handling it ourselves.

## Connecting

Now let's test the server out locally. First, we install dependencies.

```bash
virtualenv venv -p python3
source venv/bin/activate
pip install -r requirements.txt
```

Now we generate Python code from our `calculator.proto` file. This is how
we get the definitions for our `calculator_pb2` and `calculator_pb2_grpc`
modules. It's considered poor form to check these into source code, so they're
included in our `.gitignore` file.

```bash
python -m grpc_tools.protoc \
    -I. \
    --python_out=. \
    --grpc_python_out=. \
    calculator.proto
```

Finally, we start the server:

```bash
python server.py
```

Now the server should be listening on port `50051`. We'll use the tool
[`grpcurl`](https://github.com/fullstorydev/grpcurl) to manually interact with it.

```bash
grpcurl \
    --plaintext \             # Use an unencrypted connection.
    -proto calculator.proto \ # Point it at the protobuf definition.
    localhost:50051 \         # The address of the local server.
    -d '{"first_operand": 2.0, "second_operand": 3.0, "operation": "ADD"}' \
    Calculator.Calculate      # The method on the server.
```

We tell `grpcurl` where to find the protocol buffer definitions and server.
Then, we supply the request. `grpcurl` gives us a nice mapping from JSON to
protobuf. We can even supply the operation enumeration as a string. Finally, we
invoke the `Calculate` method on the `Calculator` service. If all goes well, you
should see:

```bash
{
  "result": 5
}
```

Great! We've got a working calculator server. Next, let's put it inside a
Docker container.

## Containerizing the Server

We're going to use the official Dockerhub Python 3.8 image as our base image.

```Dockerfile
FROM python:3.8
```

We'll put all of our code in `/srv/grpc/`.

```Dockerfile
ENV SRV_DIR "/srv/grpc"

RUN mkdir -p "${SRV_DIR}"

WORKDIR "${SRV_DIR}"

COPY server.py *.proto requirements.txt "${SRV_DIR}/"
```

We install our Python package dependencies into the container.


```Dockerfile
RUN pip install -r requirements.txt && \
    python -m grpc_tools.protoc \
        -I. \
        --python_out=. \
        --grpc_python_out=. \
        calculator.proto
```

Finally, we set our container up to run the server by default.

```Dockerfile
CMD ["python", "server.py"]
```

Now we can build our image. In order to deploy to Cloud Run, we'll be pushing to
the `gcr.io` container registry, so we'll tag it accordingly.

```bash
docker build -t gcr.io/GCP_PROJECT/grpc-calculator:latest
```

The tag above will change based on your GCP project name. We're calling the
service `grpc-calculator` and using the `latest` tag.

Now, before we deploy to Cloud Run, let's make sure that we've containerized our
application properly. We'll test it by spinning up a local container.

```bash
docker run -d -p 50051:50051 gcr.io/GCP_PROJECT/grpc-calculator:latest
```

If all goes well, `grpcurl` will give us the same result as before:

```bash
grpcurl \
    --plaintext \
    -proto calculator.proto \
    localhost:50051 \
    -d '{"first_operand": 2.0, "second_operand": 3.0, "operation": "ADD"}' \
    Calculator.Calculate
```

## Deploying to Cloud Run

Cloud Run needs to pull our application from a container registry, so the first
step is to push the image we just built.

Make sure that [you can use `gcloud`](https://cloud.google.com/sdk/gcloud/reference/auth/login)
and are [able to push to `gcr.io`.](https://cloud.google.com/container-registry/docs/pushing-and-pulling)

```bash
gcloud auth login
gcloud auth configure-docker
```

Now we can push our image.

```bash
docker push gcr.io/GCP_PROJECT/grpc-calculator:latest
```

Finally, we deploy our application to Cloud Run:

```bash
gcloud run deploy --image gcr.io/GCP_PROJECT/grpc-calculator:latest --platform managed
```

This command will give you a message like
```
Service [grpc-calculator] revision [grpc-calculator-00001-baw] has been deployed and is serving 100 percent of traffic at https://grpc-calculator-xyspwhk3xq-uc.a.run.app
```

We can now access the gRPC service at
`grpc-calculator-xyspwhk3xq-uc.a.run.app:443`. Go ahead and leave the `https://`
prefix off. Notice that this endpoint is secured with TLS even though the serve
we wrote is using a plaintext connection. Cloud Run provides a proxy that
provides TLS for us. We'll account for that in our `grpcurl` invocation by
leaving off the `--plaintext` flag.

```bash
grpcurl \
    -proto calculator.proto \
    grpc-calculator-xyspwhk3xq-uc.a.run.app:443 \
    -d '{"first_operand": 2.0, "second_operand": 3.0, "operation": "ADD"}' \
    Calculator.Calculate
```

And now you've got an auto-scaling calculator gRPC service!
