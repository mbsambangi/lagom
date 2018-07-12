---
layout: post
title: TDD of Microserivce-Nanoservice Combo with a Contract (In other words, SpringBoot-SpringFunction Combo with a Spring RestDocs-Contract)<img src="/assets/images/blog2/cdc.png" style="width:100%"/>
comments: False
categories:
- blog
---
---
# What is this all about?
Most of the monolithic applications developed are modularized into layers (tiers) based on their purpose like Web (Client-side), Business logic (Server-side), database access (Database-side). Slicing the application vertically based on bounded contexts into independent micro-monoliths is like a microservices architectural style.

Spring Boot & Spring Cloud Frameworks made the Development of Microservice style applications easy.

PaaS (Platform-as-a-Service) offerings like Cloud Foundry made the Operations part of microservices easy with simple configurations and deployments.

Serverless programming (sometimes referred to as Nanoservices) is gaining popularity day by day. All major cloud vendors have Serverless computing platforms. Serverless computing sometimes also called FaaS (Function-as-a-Service).

If a project is a collection of Microserivces, chances are a few of those microservices fit right as Nanoservices or a remote Nanoservice (serverless) dependency may exist. And then that project can be a combination of Microservices-Nanoservices.

TDD (Test Driven Development) is essential to tackle the complexity and communication between services. Consumer-Driven-Contracts are handy and reduce the dependency on other services during development.

From a developers point of view what matters is the programming language, development Framework, and ease of testing and deployment.

In this blog post I am trying to address the Dev part of DevOps with a simple Microserivce-to-Nanoservice style project by taking advantage of Spring Cloud Framework.

# What are all "good to know"?
Familiarity with following Spring Frameworks is good to know.
- Spring Boot & MockMvc
- Spring Cloud Function
- Spring Cloud Contract
- Spring RestDocs

I have provided links to various resources at the end of this blog post to get an understanding of these concepts.

---
# Sample Application
## Use case:

When you call a doctor's office to schedule an appointment, they first verify your health insurance to make sure you have right coverage.

## Design:

First we develop an appointment REST service at Doctor's Office application. Then a members function at fictional Health Insurance company (HealthFirst) application to verify insurance coverage.

Typical workflow is

- We call Doctor-Office's appointment request REST service by supplying First name, Last name, and Health Insurance MemberID.
- Appointment request REST service in-turn calls Health Insurance company's Member-Function to get given member's coverage level.

## Doctor-Office (Hospital) [Consumer of the Contract]

Doctor-Office is an independent Spring boot service. We expose a REST service to request an appointment. This application consumes Health Insurance's Member-Function Serverless application.
##### - [Full Source Code is here](https://github.com/mbsambangi/doctor-office)

<img src="/assets/images/blog2/consumer_uml.png" style="width:100%"/>

## Member-Function (Health Insurance Company) [Producer of the Contract]

Member-Function is a Spring Cloud Function. It provides coverage level for a given member Id.

This is the producing service which produces the service contract to check its member's coverage type.
##### - [Full Source Code is here](https://github.com/mbsambangi/member-function)
---
# Consumer-Driven-Contract initiated from Producer side

In a Consumer-Driven-Contract the contract starts from consumer side. Here Doctor-Office is the consuming application and Health Insurance companies Member-Function app is the producing application.

But, it makes more sense if Health Insurance company publishes its contract out to all associated hospitals.

<img src="/assets/images/blog2/cdc.png" style="width:100%"/>

In this sample application, lets generate a service contract at Health Insurance application. Then, publish that contract out so that any doctor's office (Hospital) would know how to check health insurance coverage.

---
# Nanoservice - Member-Function (Contract Producer)
After considering all the design aspects Health Insurance check application is better suited to be developed as a Nanoservice, aka a serverless application.

For following reasons this can be a serverless app
- Its a simple check service.
- Used by various associated hospitals via different  communication channels (Web, Messaging, events)
- Doesn't need to be up and running all the time.
- No need to gauge the load and traffic.

## Setup
Lets implement Health Insurance company's Member-Function app using Spring Cloud Function. We need following dependencies in our pom.xml to start with.

Serverless apps are invoked by events via various channels. Lets choose Web channel to keep it simple. We need spring-cloud-starter-function-web. This starter BOM provides Function implementation also exposes the function as a REST web serivce.

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-function-web</artifactId>
</dependency>
```

Then, we need RestDocs dependency for generating doc snippets.

```
<dependency>
	<groupId>org.springframework.restdocs</groupId>
	<artifactId>spring-restdocs-mockmvc</artifactId>
	<scope>test</scope>
</dependency>
```
And, we need ASCII Doc dependency to generate html out of doc snippets.

```
<dependency>
	<groupId>org.springframework.restdocs</groupId>
  <artifactId>spring-restdocs-asciidoctor</artifactId>
</dependency>
```
To integrate Rest Docs with Cloud Contract we need following WireMock dependency.

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-contract-wiremock</artifactId>
  <scope>test</scope>
</dependency>
```
Spring-cloud-starter-contract-verifier BOM is needed if we are writing a DSL contract manually. We don't need this here as we are not going to write DSL contract. Rest Docs auto generate the DSL contract for us.

Now, we need following plugin to assemble generated contract and stubs to publish out to consuming applications. To demonstrate, we install the stubs jar to local Maven repository.

```
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
  <configuration>
	<attach>true</attach>
	<descriptors>
  	<descriptor>${project.basedir}/src/assembly/stub.xml</descriptor>
  </descriptors>
  </configuration>
  <executions>
  	<execution>
  		<id>stub</id>
  		<phase>prepare-package</phase>
			<goals>
    		<goal>single</goal>
    	</goals>
    	<inherited>false</inherited>
    </execution>
  </executions>
</plugin>
```
and the stub.xml which defines the assembly rules. Please refer the codebase for the stub.xml file.

We don't need spring-cloud-contract-maven-plugin as we are not going to generate test-case out of DSL contract. Because, we are going to auto generate DSL from a test-case.

## TDD
Lets start with a failing test-case. We use MockMvc to call the function which I am going to describe in a bit.

So, we invoke the function by passing in a HealthFirstMember object. Member ID is mandatory field and other fields are optional.

After invoking the Mock service we check if the coverage is MEDICAL to pass the test-case.

{% highlight java %}
@Test
public void provideCoverageForGivenMemberId() throws Exception {
	HealthFirstMember member = new HealthFirstMember();
	member.setMemberId("123456789");
	member.setCoverage(HealthFirstMember.Coverage.NONE);

	MvcResult result = mockMvc.perform(post("/members")
	.contentType(MediaType.APPLICATION_JSON_UTF8)
	.content(json.write(member).getJson())
	).andReturn();

	mockMvc.perform(asyncDispatch(result))
	.andExpect(status().isOk())
	.andExpect(jsonPath("coverage").value("MEDICAL"))
}
{% endhighlight %}

We will revisit the test-case when we are ready to generate contracts and docs.

## Member Function
In order to pass the test-case we need to real implementation of a Function. Following code snippet shows how this is implemented using Spring Cloud Function.

{% highlight java %}
	public static void main(String[] args) {
		SpringApplication.run(MemberFunctionApplication.class, args);
	}

	@Bean
	public Function<HealthFirstMember, HealthFirstMember> members() {
		return member -> {
			   member.setCoverage(HealthFirstMember.Coverage.MEDICAL);
			   return member;
	  };
	}
{% endhighlight %}

Now, if you run the test-case it should pass.

<img src="/assets/images/blog2/function-test-pass.png" style="width:100%"/>

## Contract Generation
Now we have a passing test-case and implementation. We are ready to generate a contract and stubs out of it.

Lets re-visit the test-case to achieve this.

Lets add following annotation so Rest Docs can generate snippets.

{% highlight java %}
@AutoConfigureRestDocs(outputDir = "target/snippets")
{% endhighlight %}

Then, we can call WireMockRestDocs.verify() method to register and check request/response. Then call the .stub() to store reqeust/response stubs.

{% highlight java %}
mockMvc.perform(asyncDispatch(result))
        .andExpect(status().isOk())
        .andExpect(jsonPath("coverage").value("MEDICAL"))
        .andDo(WireMockRestDocs.verify().jsonPath("$.memberId")
        .contentType(MediaType.APPLICATION_JSON_UTF8).stub("healthfirst-member-check"))
{% endhighlight %}

Finally, we call SpringCloudContractRestDocs.dslContract() by passing it to MockMvcRestDocumentation.document() to generate the DSL contract.

After, all the above steps the completed test-case looks like this.

{% highlight java %}
public void provideCoverageForGivenMemberId() throws Exception {
        HealthFirstMember member = new HealthFirstMember();
        member.setMemberId("123456789");
        member.setCoverage(HealthFirstMember.Coverage.NONE);

        MvcResult result = mockMvc.perform(post("/members")
        .contentType(MediaType.APPLICATION_JSON_UTF8)
        .content(json.write(member).getJson())
        ).andReturn();

        mockMvc.perform(asyncDispatch(result))
        .andExpect(status().isOk())
        .andExpect(jsonPath("coverage").value("MEDICAL"))
        .andDo(WireMockRestDocs.verify().jsonPath("$.memberId")
        .contentType(MediaType.APPLICATION_JSON_UTF8).stub("healthfirst-member-check"))
        .andDo(MockMvcRestDocumentation.document("healthfirst-member-check",
                SpringCloudContractRestDocs.dslContract()));
    }
{% endhighlight %}

Now, run
```
$mvn clean install
```
and check the target folder. Generated snippets, docs, json stub, and dsl groovy contract should be available under snippets folder.

<img src="/assets/images/blog2/target-snippets.png" style="width:100%"/>

The stubs.jar is installed to local Maven repository. Which serves as the contract and shared with consuming application.

<img src="/assets/images/blog2/function-stub.png" style="width:100%"/>

---
# Microserivce - Doctor-Office - (Contract Consumer)
Lets implement this consuming application a simple Spring boot service with a simple REST service. This application calls our Nanoservice Member-Function above to check a member's coverage.

As we are doing TDD and offline development we need code against to the shared contract.

## Setup
We need following dependencies in our pom.xml to start with.

We are going to write a REST controller we need spring-boot-starter-web.

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

To call the Health Insurance company's Member-Function we can use either RestTemplate or Feign. Lets use Feign. So, we need Feign dependency for defining a Feign Client Interface

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
Now, we need contract stub runner to run the stub shared by Health Insurance company.

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
</dependency>
```
Finally, we need the Member-Function stub dependency listed as dependency. Following dependency gets the -stubs.jar which we installed to local maven repository before.

```
<dependency>
	<groupId>com.healthfirst</groupId>
	<artifactId>member-function</artifactId>
	<classifier>stubs</classifier>
	<version>0.0.1-SNAPSHOT</version>
	<scope>test</scope>
	<exclusions>
		<exclusion>
			<groupId>*</groupId>
			<artifactId>*</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```
## TDD
Lets write a failing test-case for the REST controller. We use MockMvc to call the controller which I am going to describe in a bit.

So, we invoke the controller by passing in a Appointment object. We need Firstname, Lastname, and memberId.

After invoking the Mock service we check if the appointment is CONFIRMED to pass the test-case.

{% highlight java %}
@Test
public void scheduleAppointmentWhenPatientHasCoverage() throws Exception {
		Appointment appointment = new Appointment();

		appointment.setFirstName("Madhu");
		appointment.setLastName("Sambangi");
		appointment.setMemberId("123456789");
		appointment.setDateOfBirth("01/01/2018");

		mockMvc.perform(post("/api/v1/appointments")
		.contentType(MediaType.APPLICATION_JSON_UTF8)
		.content(objectMapper.writeValueAsString(appointment))
		.accept(MediaType.APPLICATION_JSON_UTF8))
						.andDo(print()).andExpect(status().isOk())
		.andExpect(jsonPath("status").value("CONFIRMED"));
}
{% endhighlight %}

We will revisit the test-case when we are ready to generate contracts and docs.

## Appointments REST Service
We need the controller implemented to pass the test-case. Also, inject the HealthFirstService Feign client so that it can call Member-Function.

{% highlight java %}
@RestController
@RequestMapping("api/v1")
public class PatientController {

	private HealthFirstService healthFirstService;

	public PatientController(HealthFirstService healthFirstService) {
			this.healthFirstService = healthFirstService;
	}

	@PostMapping("/appointments")
	public AppointmentResponse appointments(@RequestBody Appointment appointment) {
			AppointmentResponse response = new AppointmentResponse();
			HealthFirstMember member = new HealthFirstMember();
			member.setMemberId(appointment.getMemberId());

			member = healthFirstService.verifyCoverage(member);

			if (member.getCoverage() == HealthFirstMember.Coverage.MEDICAL) {
					response.setStatus(AppointmentResponse.AppointmentStatus.CONFIRMED);
			}

			return response;
	}
}
{% endhighlight %}

## Feign Client to call Member Function
And the Feigin client to call the Health Insurance's Memebr-Function service.

{% highlight java %}
@FeignClient(name = "HealthFirstService",
    url = "http://localhost:8080", fallback = HealthFirstService.HealthFirstServiceFallback.class)
public interface HealthFirstService {

    @RequestMapping(method = RequestMethod.POST, path = "/members")
    @Headers("Accept:application/json;charset=UTF-8")
    HealthFirstMember verifyCoverage(@RequestBody HealthFirstMember member);

    @Component
    class HealthFirstServiceFallback implements HealthFirstService {
        @Override
        public HealthFirstMember verifyCoverage(@RequestBody HealthFirstMember member) {
            member.setCoverage(HealthFirstMember.Coverage.NONE);
            return member;
        }
    }
}
{% endhighlight %}

Now, if you run the test-case it still fails complaining 'Connection Refused' error. Because Feign client tries to call the Member-Function at http://localhost:8080. We haven't configured the stub runner in the test case to mock the Member-Function using the contract yet.

## Stubs from Member-Function Contract
Lets re-visit the test-case to configure the stub so that Feign client will work.

Lets add following annotation in Test class so the stub will run at localhost port 8080.

{% highlight java %}
@AutoConfigureStubRunner(ids = "com.healthfirst:member-function:+:stubs:8080", stubsMode = StubRunnerProperties.StubsMode.LOCAL)
{% endhighlight %}

## Appointments Test
Now the test case passes covering end-to-end testing.
- Calling the appointment REST controller First

```
MockHttpServletRequest:
      HTTP Method = POST
      Request URI = /api/v1/appointments
       Parameters = {}
          Headers = {Content-Type=[application/json;charset=UTF-8], Accept=[application/json;charset=UTF-8]}
             Body = {"firstName":"Madhu","lastName":"Sambangi","memberId":"123456789","dateOfBirth":"01/01/2018"}
    Session Attrs = {}

Handler:
             Type = com.hospital.doctoroffice.PatientController
           Method = public com.hospital.doctoroffice.AppointmentResponse com.hospital.doctoroffice.PatientController.appointments(com.hospital.doctoroffice.Appointment)
```

- Which calls the Feign client for member's coverage.

```
127.0.0.1 - POST /members

User-Agent: [Java/1.8.0_131]
Connection: [keep-alive]
Host: [localhost:8080]
Accept: [*/*]
Content-Length: [40]
Content-Type: [application/json;charset=UTF-8]
{"memberId":"123456789","coverage":null}
```

- Feign client calls the stubbed Health Insurance's Member-Function and gets the member's coverage as per the contract.

```
Matched response definition:
{
  "status" : 200,
  "body" : "{\"memberId\":\"123456789\",\"coverage\":\"MEDICAL\"}",
  "headers" : {
    "Content-Type" : "application/json;charset=UTF-8"
  }
}
```

- PatientController checks the coverage and returns CONFIRMED if the coverage is MEDICAL.

```
MockHttpServletResponse:
           Status = 200
    Error message = null
          Headers = {Content-Type=[application/json;charset=UTF-8]}
     Content type = application/json;charset=UTF-8
             Body = {"status":"CONFIRMED"}
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
```

- Finally, the test case passes if the appointment status is CONFIRMED.

<img src="/assets/images/blog2/microservice-test.png" style="width:100%"/>

---
# Summary
This article attempts to demonstrate how powerful is Test Driven Development & Consumer-Driven-Contract when there is a dependency on an internal/external service. If there is a mix of microservices and nanoservices this approach really boosts developer productivity to produce quality testable code.

Please go over the Resources & References section below to learn more of the various pieces of the Spring Framework I used in the sample application, also to understand more about the keywords like microservices, nanoservices, and serverless.

---

# Resources & References
##### - [Microserivces & Nanoservices with Java](https://www.slideshare.net/ewolff/nanoservices-and-microservices-with-java)
##### - [Spring Cloud Contract with RestDocs](http://cloud-samples.spring.io/spring-cloud-contract-samples/tutorials/rest_docs.html)
##### - [Spring Cloud Function](https://github.com/spring-cloud/spring-cloud-function)
##### - [Serverless Architecture](https://martinfowler.com/articles/serverless.html)
---
