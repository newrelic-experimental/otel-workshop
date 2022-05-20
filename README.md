# OpenTelemetry on the Cloud

The goal of this workshop is to see first hand the power of OpenTelemetry for cloud native applications

## Prerequisites

[**Install Minikube**](https://minikube.sigs.k8s.io/docs/start/) 

[**Install kubectl**](https://kubernetes.io/docs/tasks/tools/)

[**Install Skaffold**](https://skaffold.dev/)

**From a terminal with administrator access (but not as root), run** 

```bash
minikube start --memory 8192 --cpus 6
```

*This application is resource heavy so get ready for some 💨*

**Check that your cluster is up and running by running**

```bash
kubectl get nodes
```

**Clone the repo for the sample application** 

```bash
git clone https://github.com/newrelic-experimental/otel-workshop.git
```

**Export env variables** 

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp.nr-data.net:4317
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=https://otlp.nr-data.net:4317
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=https://otlp.nr-data.net:4317
export OTEL_EXPORTER_OTLP_HEADERS=api-key=<NEWRELIC_INGEST_LICENSE_KEY>
export NEW_RELIC_API_KEY=<NEWRELIC_INGEST_LICENSE_KEY>
export NEW_RELIC_LICENSE_KEY=<NEWRELIC_INGEST_LICENSE_KEY>
export NEW_RELIC_HOST=collector.newrelic.com
```

**Run Skaffold**

*Note that this step will take ~15-20 minutes the first time you run this command because Docker has to build and push each of the microservices*

```bash
skaffold dev
```

## Adding a delay

Your first mission is to add an artificial delay in one of the functions to see the full power of OpenTelemetry and distributed tracing. Let’s say we want to know what exactly causes a delay in the frontend? Distributed tracing makes it easy for you to follow the journey of a request as it travels throughout your system.  

![Screen Shot 2022-05-15 at 5.02.10 PM.png](images/Screen_Shot_2022-05-15_at_5.02.10_PM.png)

<aside>
📜 **In `main.go`, add a 0.1sec delay for the `createQuoteFromCount` and a 0.33 sec delay for `CreateQuoteFromFloat`functions.**

When you head into New Relic distributed tracing you should see something like this for your Distributed trace with your artificial delay clearly visible.

![Screen Shot 2022-05-15 at 5.27.50 PM.png](images/Screen_Shot_2022-05-15_at_5.27.50_PM.png)

- **🙈 Solution**
    
    ```go
    func CreateQuoteFromCount(value float64) Quote {
    	...
    	time.Sleep(time.Second / 10)
    	...
    }
    ```
    
    ```go
    func CreateQuoteFromFloat(value float64) Quote {
    	...
    	time.Sleep(time.Second / 3)
    	...
    }
    ```
    
</aside>

## **Building spans**

However, let’s say we want to get one level deeper and want to see what caused the spike in the application. You are able to add custom spans to 

<aside>
📜 **In the `GetQuote`function in `main.go`, build individual spans for the `createQuoteFromCount` and `CreateQuoteFromFloat`functions**

![Screen Shot 2022-05-15 at 5.44.23 PM.png](images/Screen_Shot_2022-05-15_at_5.44.23_PM.png)

- **🙈 Solution**
    
    ```go
    func CreateQuoteFromCount(value float64 , ctx context.Context) Quote {
    	ctx, childSpan := tracer.Start(ctx, "CreateQuoteFromCount")
    	defer childSpan.End()
    	...
    }
    ```
    
    ```go
    func CreateQuoteFromFloat(value float64 , ctx context.Context) Quote {
    	ctx, childSpan := tracer.Start(ctx, "CreateQuoteFromFloat")
    	defer childSpan.End()
    	...
    }
    ```
    
    ```go
    func (s *server) GetQuote(ctx context.Context, in *pb.GetQuoteRequest) (*pb.GetQuoteResponse, error) {
    		// 1. Generate a quote based on the total number of items to be shipped.
    	quote := CreateQuoteFromCount(0, ctx)
    }
    ```
    
</aside>

## Adding **span attributes**

Let’s say that you wanted to add more context into the traces so you can get more business insights out of your data.

Attributes are keys and values that are applied as metadata to your spans and are useful for aggregating, filtering, and grouping traces. [Attributes](https://opentelemetry.io/docs/instrumentation/go/manual/#span-attributes) can be added at span creation, or at any other time during the lifecycle of a span before it has completed. 

<aside>
📜 **In the `ShipOrder`function in `main.go`, add code to attach the `state`, `zipcode` , and `city` attributes to each `shipOrder` span you created in the previous step!**

> When you are finished you should be able to see the attributes you added when you click on the shipOrder span → attributes tab on the right panel.
> 

![Screen Shot 2022-05-15 at 2.59.42 PM.png](images/Screen_Shot_2022-05-15_at_2.59.42_PM.png)

> You should be able to run the `NRQL` query 
`SELECT count(*) FROM Span WHERE entity.name='shippingservice' FACET state`
to get the breakdown of all orders processed by state
> 

![Screen Shot 2022-05-15 at 5.38.11 PM.png](images/Screen_Shot_2022-05-15_at_5.38.11_PM.png)

- **🙈 Solution**
    ```go
    import (
        ...
        "go.opentelemetry.io/otel/attribute"
    }
    ```
    
    ```go
    func (s *server) ShipOrder(ctx context.Context, in *pb.ShipOrderRequest) (*pb.ShipOrderResponse, error) {
    ...
    	parentSpan.SetAttributes(
    		attribute.String("address", baseAddress), 
    		attribute.String("city", in.Address.City), 
    		attribute.String("state", in.Address.State))
    ****}
    ```
    
</aside>

## Adding Errors

Unlike system exceptions, application exceptions allow you to bubble up application-level activity that might cause errors, for example, invalid input argument values to a business method. 

<aside>
📜 **In the `ShipOrder`function in `main.go`, add some logic to throw an error when a zip code is not a 5 digit number. It should show up like the screenshot below in NR1.**

![Screen Shot 2022-05-15 at 5.46.42 PM.png](images/Screen_Shot_2022-05-15_at_5.46.42_PM.png)

- **🙈 Solution**
    
    ```go
    func (s *server) ShipOrder(ctx context.Context, in *pb.ShipOrderRequest) (*pb.ShipOrderResponse, error) {
    ...
    **if(in.Address.ZipCode < 10000 || in.Address.ZipCode > 99999){
           parentSpan.SetStatus(1, "zipcode is invalid") 
       }**
    }
    ```
    
</aside>

### Setting the status for span

Following the [OTel specification for the Tracing API](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/api.md#statuscanonicalcode) , SetStatus sets the status of the Span in the form of a code and a description, overriding previous values set. 

The optional `Description` field provides a descriptive message of the `Status`. 

`Description` MUST only be used with the `Error` `StatusCode` value. An empty `Description` is equivalent with a not present one.

```glsl
SetStatus(code codes.Code, description string)
```

### What to set a status code

`**StatusCode` is one of the following values:**

`Unset` (code = 0 or unset) The default status.

`Ok` (code  = 2) The operation has been validated by an Application developer or Operator to have completed successfully.

`Error` (code = 1) The operation contains an error.