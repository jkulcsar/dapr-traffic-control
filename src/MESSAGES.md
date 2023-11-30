
## Messages

Hello all, went through a few of the simple examples on how to use dapr, including the traffic control example (https://github.com/EdwinVW/dapr-traffic-control)
Works great out of the box so I thought I'll try to switch pubsub from RabbitMQ to NATS Jetstream.
NATS (as a container) is started, works.
Added a `pubsub.nats.yaml` file with the Following content:
```
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub.mm
  namespace: dapr-trafficcontrol
spec:
  type: pubsub.jetstream
  version: v1
  metadata:
  - name: natsURL
    value: nats://localhost:4222 # Required
  - name: name
    value: "my-conn-name"
  - name: streamName
    value: "my-stream"
  - name: durableName 
    value: "my-durable-subscription"
  - name: queueGroupName
    value: "my-queue-group"
scopes:
  - trafficcontrolservice
  - finecollectionservice
```
NATS is initialized well however I'm seeing the following error:
```
== APP == fail: TrafficControlService.Controllers.TrafficController[0]
== APP ==       Error occurred while processing EXIT
== APP ==       Dapr.DaprException: Publish operation failed: the Dapr endpoint indicated a failure. See InnerException for details.
== APP ==        ---> Grpc.Core.RpcException: Status(StatusCode="Internal", Detail="error when publish to topic speedingviolations in pubsub pubsub.mm: nats: no response from stream")
== APP ==          at Dapr.Client.DaprClientGrpc.MakePublishRequest(String pubsubName, String topicName, ByteString content, Dictionary`2 metadata, String dataContentType, CancellationToken cancellationToken)
== APP ==          --- End of inner exception stack trace ---
== APP ==          at Dapr.Client.DaprClientGrpc.MakePublishRequest(String pubsubName, String topicName, ByteString content, Dictionary`2 metadata, String dataContentType, CancellationToken cancellationToken)
== APP ==          at TrafficControlService.Controllers.TrafficController.VehicleExitAsync(VehicleRegistered msg, DaprClient daprClient) in C:\Users\jk\source\repos\dapr\dapr-traffic-control\src\TrafficControlService\Controllers\TrafficController.cs:line 97
time="2023-11-26T16:43:57.6934122+01:00" level=error msg="Failed processing MQTT message: trafficcontrol/exitcam/0: fails to send binding event to http app channel, status code: 500 body: {\"type\":\"https://tools.ietf.org/html/rfc7231#section-6.6.1\",\"title\":\"An error occurred while processing your request.\",\"status\":500,\"traceId\":\"00-ffef56b8f962fbe119daeabd79587d6d-25d4a16dcded12ad-01\"}" app_id=trafficcontrolservice component="exitcam (bindings.mqtt/v1)" instance=E7240 scope=dapr.contrib type=log ver=1.12.2
```
I'm not even sure the error has anything to do with NATS as PubSub. In the example, the Simulator is sending events using MQTT. I'm at a bit of a loss on how to continue.
Any help or suggestions would be much appreciated. Thank you!

## Solutions

When using NATS JetStream, you need to create a stream and a consumer before you can publish messages to it. The Dapr component for NATS JetStream does not create these for you, so you need to create them yourself. You can do this using the [NATS CLI](https://docs.nats.io/jetstream/nats_tools/jsm/jsm) or the [NATS Management UI](https://docs.nats.io/jetstream/nats_tools/jetstream_management_ui).

```bash
nats -s localhost:4222 stream add traffic-control-stream --subjects=speedingviolations --subjects=deadletters
```

TODO: verify the following command is correct (need to check the NATS docs and confirm if when a consumer is created, it automatically creates the stream if it doesn't exist)

```bash
nats -s localhost:4222 consumer add traffic-control-stream --stream=traffic-control-stream --ack=explicit --deliver=all --replay=instant --filter subject=deadletters --filter stream=traffic-control-stream --durable=traffic-control-consumer
```

## References to answer some of the questions in the Solutions section

https://docs.nats.io/nats-concepts/jetstream/consumers
https://docs.dapr.io/reference/components-reference/supported-pubsub/setup-jetstream/#example-competing-consumers-pattern
