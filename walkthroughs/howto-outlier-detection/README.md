
# Configuring Outlier Detection

  

This walkthrough demonstrates the usage of App Mesh's Outlier Detection feature. Clone this repo and navigate to `aws-app-mesh-examples/walkthroughs/howto-outlier-detection` to get started!

## Introduction

Outlier Detection is a form of passive health check that temporarily ejects an endpoint/host of a given service (represented by a Virtual Node) from the load balancing set when it meets some failure threshold (hence considered an *outlier*). App Mesh currently supports the definition of an outlier using the number of server errors (any 5xx Http response) a given endpoint has returned within a given time interval. With Outlier Detection, intermittent failures caused by degraded hosts can be mitigated as the degraded host is no longer a candidate during load balancing.

Some may also recognize this design pattern under the term circuit breaking. An ejected endpoint is eventually returned to the load balancing set, and each time the same endpoint gets ejected, the longer it stays ejected.

In the App Mesh API, we define Outlier Detection on the server side; that is, the service defines the criteria for its hosts to be considered an outlier. Therefore, Outlier Detection should be defined in the server Virtual Node alongside active health checks. 

Here are the necessary fields to configure Outlier Detection:

-  **Interval**: the time between each outlier detection sweep. During the sweep is when hosts get ejected or un-ejected.

-  **MaxServerErrors**: the threshold for the number of server errors returned by a given endpoint during an outlier detection interval. If the server error count is greater than or equal to this threshold the host is ejected. A server error is defined as any HTTP 5xx response (or the equivalent for gRPC and TCP connections).

-  **BaseEjectionDuration**: The amount of time an outlier host is ejected for is <baseEjectionDuration> * number of times this specific host has been ejected. For example, if baseEjectionDuration is 30 seconds, an outlier host A would first be ejected for 30 seconds and returned to the load balancing set on the next sweep. If host A later gets ejected again, host A will be removed from the load balancing set for 30 seconds * 2 (this host is being ejected the second time) = 1 minute.

-  **MaxEjectionPercent**: The threshold for the max percentage of outlier hosts that can be ejected from the load balancing set. maxEjectionPercent=100 means outlier detection can potentially eject all of the hosts from the upstream service if they are all considered outliers, leaving the load balancing set with zero hosts. In reality, due to a default panic behavior in Envoy, if more than 50% of the endpoints behind a service are considered outliers or are failing health checks, the outlier detection ejection is overturned and traffic will be served to these degraded endpoints.

## Step 1: Prerequisites

We will need the latest version of the App Mesh Preview CLI for this walkthrough. You can download and use the latest version using the command below.
```bash

aws configure add-model \

--service-name appmesh-preview \

--service-model https://raw.githubusercontent.com/aws/aws-app-mesh-roadmap/master/appmesh-preview/service-model.json

```
In addition, this walkthrough makes use of the unix command line utility `jq`. If you don't already have it, you can install it from [here](https://stedolan.github.io/jq/).

Finally, to generate traffic and observe the server responses, we will leverage the open source Http load testing tool [vegeta](https://github.com/tsenart/vegeta). You can choose to install it locally or run the commands we will use later within a Docker container using this [image](https://hub.docker.com/r/peterevans/vegeta/).

 
## Step 2: Set Environment Variables

We need to set a few environment variables before provisioning the infrastructure. 

```bash
export AWS_ACCOUNT_ID=<account id>

export AWS_DEFAULT_REGION=us-west-2

export ENVOY_IMAGE=<get the latest from https://docs.aws.amazon.com/app-mesh/latest/userguide/envoy.html>
```

## Step 3: Set up our Service Mesh
The mesh configuration for this walkthrough is rather straightforward:
Our mesh contains two virtual nodes, `front-node` and `color-node`. The `front-node` is backed by a virtual service `color.howto-outlier-detection.local` that is provided by the `color-node`. 

```bash
./mesh.sh up
```
## Step 4: Deploy Infrastructure and Service
  We'll build the frontend and color applications under `src` into Docker images, create and push to the ECR repo under the account `AWS_ACCOUNT_ID`, then deploy CloudFormation stacks for network infrastructure and the ECS services. 
  
```bash
./deploy.sh
```
The output of the application CloudFormation stack should return an ALB endpoint in the format
```http://<foo>.us-west-2.elb.amazonaws.com```. We'll use this to reach our frontend service. For convenience let's export it:
```bash
export ALB_ENDPOINT=<>
```

Note that the applications use go modules. If you have trouble accessing https://proxy.golang.org during the deployment you can override the `GOPROXY` by setting `GO_PROXY=direct`, i.e. run
```bash
GO_PROXY=direct ./deploy.sh
```
instead.

## Step 5: Before Enabling Outlier Detection
In this walkthrough, the frontend service calls the color service to get a color via `/get`. Under normal circumstances, the color service always responds with the color purple (because it's a nice color). 

In addition, the frontend service is able to inject faults to the color service by making a request to `/fault`. When a color service server receives this request, it will start returning 500 internal service error on `/get`. The fault can be recovered via `/recover` . 

Finally, the frontend service keeps track of each unique host behind the color service and a counter of the response statuses they've returned. These stats can be retrieved via `/stats` and reset with `reset_stats` on the frontend service.

Let's start by issuing a simple get color request:
```bash
curl -i $ALB_ENDPOINT/color/get
```
Let's check out the stats recorded in the frontend service:
```bash
curl -i $ALB_ENDPOINT/stats
```
We should see a single entry consisting of a HostUID and counter for its StatusOk (200), StatusError (500), and Total responses, e.g.
`[{"HostUID":"546da12e-c0b2-4c2a-9615-b1d9590d5a43","Counter":{"StatusOk":1,"StatusError":0,"Total":1}}]`

Now, let's generate more traffic to frontend service using vegeta. By default the request rate is 50/sec, so with duration=4s we'd be sending 200 requests.
```bash
echo "GET $ALB_ENDPOINT/color/get" | vegeta attack -duration=4s | tee results.bin | vegeta report
```
Observe the output. In particular the Status Codes row should show 200:200 (status code:request count).

Observe the frontend stats; we can see that there are four HostUIDs representing the four color service tasks and each host should be serving ~50 requests. This is because we have four endpoints in the load balancing set and by default a round-robin load balancing strategy is enacted.
```bash
curl -i $ALB_ENDPOINT/stats
<blah>
```

Finally, let us inject a fault to one of the color service hosts. We want one of the four to be returning 500.
```bash
curl -i $ALB_ENDPOINT/color/fault
```
Note that the response also returns the hostUID identifying the host. 

Now let us issue 200 requests to the frontend service again:
```bash
echo "GET $ALB_ENDPOINT/color/get" | vegeta attack -duration=4s | tee results.bin | vegeta report
```
The status code distribution should now look something like 200:150 500:50. If we check the frontend stats again, we should see that one of the four hosts now have non-zero StatusError and all the math should add up. 

## Step 6: Outlier Detection in Action
Let's see how Outlier Detection can help us reduce the number of server errors given that one of the color hosts is degraded. Update the `color-node` virtual node with the spec `mesh/color-vn-with-outlier-detection.json`:
```bash
./mesh.sh add-outlier-detection
```
Once this update is propagated all the way down to the front-node's Envoy (give it a minute or so), let's try to issue 200 requests yet again:
```bash
echo "GET $ALB_ENDPOINT/color/get" | vegeta attack -duration=4s | tee results.bin | vegeta report
```
We should see something like:
`Status Codes  [code:count]                      200:195  500:5`
Compare this to when we didn't have Outlier Detection: a fourth of the requests were routed to the degraded host, whereas in this instance after five server errors were returned by the degraded host, Outlier Detection ejected the host from the load balancing set and traffic is no longer routed to that host. You can verify this via the frontend stats by inspecting the counters of each host. 

The `baseEjectionDuration` is configured as 10 seconds for the `color-node`, which is a relatively short amount of time. Try sending another 200 requests. If that yielded another `Status Codes  [code:count]                      200:195  500:5`, then it's likely the degraded host has already been un-ejected. Immediately send another 200 requests and you should see all requests returning 200. 
If this degraded host continues to behave like how it is, the ejection time will continue to multiply and it becomes less likely that the host would be available in the load balancing set.

The basic walkthrough ends here and albeit the example somewhat contrived (having a host *always* return 500), should have demonstrated how Outlier Detection can help in mitigating intermittent server errors caused by degraded hosts and improve service availability. 

Now for something more  **advanced**. The following section is optional and serves to demonstrate a default Envoy behavior that may affect Outlier Detection. 

### Panic and Ignore Outlier Detection
So what happens if all of the hosts were to be ejected all at once? There wouldn't be any valid endpoints to route traffic to and the service would be 0% available. Enters Envoy's [panic threshold](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/panic_threshold#arch-overview-load-balancing-panic-threshold) configuration- this value defines when to place a service into panic mode where requests will be routed to all of the hosts regardless of their health or outlier detection status. This value is set to 50 by default; in other words if more than 50 percent of the hosts were to be ejected as an outlier, then all the outliers would still be valid candidates to serve traffic! In most cases this behavior is more preferable over not trying to route requests to the service at all. Let's see it in action.

Remember that we currently have one host that continues to return 500 on `/get`. Let's fault another host:
```bash
curl -i $ALB_ENDPOINT/color/fault
```
Since we can't target this request to a specific host, observe the response that includes the HostUID to ensure the request reached a different host than the existing one. You can check the host stats through the frontend service and compare if the returned HostUID doesn't already have non-zero `StatusErrors`. If it's the same host just send the request again. 

Now let's generate traffic:
```bash
echo "GET $ALB_ENDPOINT/color/get" | vegeta attack -duration=4s | tee results.bin | vegeta report
```
We expect the newly degraded host to trigger Outlier Detection, while the existing one should also be ejected again if it is already un-ejected.

Confirm that sending another 200 requests should now all yield 200 status responses as both outliers get ejected:
```bash
echo "GET $ALB_ENDPOINT/color/get" | vegeta attack -duration=4s | tee results.bin | vegeta report
```

Finally, let's breach the panic threshold by faulting a third host using the same approach above: 
```bash
curl -i $ALB_ENDPOINT/color/fault
```
Observe that when we now send 200 requests to the service, which has 75% of its hosts returning 500, panic mode should kick in:
```bash
echo "GET $ALB_ENDPOINT/color/get" | vegeta attack -duration=4s | tee results.bin | vegeta report
```
We should expect the distribution to be roughly `200:50 500:150` since all four hosts are now serving traffic again regardless of their outlier status.

  
## Step 8: Clean Up
All resources created in this walkthrough can be deleted via:
```bash
./deploy.sh delete && ./mesh.sh down
```
