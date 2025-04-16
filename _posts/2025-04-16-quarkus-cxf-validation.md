---
layout: post
title: "Disabling WSDL Validation in Quarkus CXF"
---

When working with SOAP services using Quarkus CXF, you might encounter situations where the incoming messages don't strictly adhere to the WSDL definition. This can happen when the API evolves and new fields are added without prior notification, leading to validation errors and preventing your application from processing these messages.

By default, CXF performs validation against the WSDL schema. While this is generally a good practice, it can become problematic in dynamic environments where the API contract might not always be perfectly aligned with the deployed WSDL.

This blog post demonstrates how to disable incoming message validation in your Quarkus CXF client to handle such scenarios gracefully.

## The Problem: Strict WSDL Validation

Imagine your Quarkus application consumes a SOAP service defined by a WSDL. If the service provider adds a new, optional field to their response that isn't described in the current WSDL your client has, CXF's default validation will likely throw an exception, preventing you from processing the response.

While ideally, you would update your client's WSDL and generated code whenever the service changes, this might not always be feasible or immediately possible. In such cases, temporarily disabling validation can provide a workaround.

## The Solution: A Custom Interceptor

We can create a custom CXF interceptor to disable the JAXB validation event handler, which is responsible for enforcing the schema compliance. Here's the Java code for the interceptor:

```java
import org.apache.cxf.message.Message;
import org.apache.cxf.phase.AbstractPhaseInterceptor;
import org.apache.cxf.phase.Phase;
import org.apache.cxf.interceptor.Fault;
import io.quarkus.logging.Log;

public class DisableMessageValidationInterceptor extends AbstractPhaseInterceptor<Message>
{
    public DisableMessageValidationInterceptor()
    {
        super(Phase.PRE_PROTOCOL);
    }

    public void handleMessage(Message message) throws Fault
    {
        message.put("set-jaxb-validation-event-handler", "false");
    }
}
```

## Explanation:

We create a class DisableMessageValidationInterceptor that extends AbstractPhaseInterceptor<Message>.
The constructor calls the superclass constructor with Phase.PRE_PROTOCOL. This ensures that our interceptor runs early in the interceptor chain, before the protocol-specific processing.
The handleMessage method is where the core logic resides.
We use message.put("set-jaxb-validation-event-handler", "false"); to instruct CXF to disable the JAXB validation event handler for the current message. This effectively tells the JAXB unmarshaller to ignore validation errors.

## Integrating the Interceptor in Quarkus

To use this interceptor with your Quarkus CXF client, you need to register it in your application.properties file. Assuming your CXF client is named "my-soap-client", you would add the following line:

```java
quarkus.cxf.client."my-soap-client".in-interceptors=com.example.DisableMessageValidationInterceptor
```

## Conclusion

By implementing and registering this custom interceptor, you can effectively disable incoming message validation for your Quarkus CXF client. This can be a useful strategy when dealing with SOAP APIs that might evolve without strict adherence to the published WSDL.

However, it's crucial to understand the implications of disabling validation:

* You lose the guarantee that incoming messages strictly conform to the WSDL.
* Your application might need to be more resilient and handle unexpected data or missing fields.
* Disabling validation should be considered a temporary workaround. Ideally, you should strive to keep your client's WSDL and generated code up-to-date with the service definition.

Use this technique judiciously and ensure your application is prepared to handle potentially non-compliant messages.
