---
featured: false
title: "Micro rule-engine with Spring and Drools"
tags: [springboot, oas, kotlin, drools, rule-engine, jib, docker]
date: 2019-05-25T14:24:10+10:00
lastmod: 2019-05-25T14:24:10+10:00
draft: false
---

![Drools is cool](/img/posts/2019-05-25-drools.png 'Drools is cool')

It is 2019, while every man and his dog talks about Machine Learning & AI, I have spent some time this week to play with Drools (HRS - Hybrid Reasoning system) and built an API with it. It is interesting to know that during the well-known ["AI Winter"](https://en.wikipedia.org/wiki/AI_winter), Expert systems have been quite popular commercially in Business Rules Management System (BRMS) market, namely Red Hat JBoss and IBM JRules. If you love reading more, then this is [a good start](https://docs.jboss.org/drools/release/7.22.0.Final/drools-docs/html_single/index.html#_hybridreasoningchapter).

Whether such systems are useful or not is hard to say and depends on business scenarios; Martin Fowler detailed in [his blog](https://martinfowler.com/bliki/RulesEngine.html) dated in 2009 that one should keep a few things in mind when designing rule systems. I just summarise them here:

- Too many rules lead to more sophisticated algorithms, harder for reasoning by user
- With chaining, interaction of rule can be complex
- Define rules declaratively is easy to read, but debugging can be a problem (imperative vs declarative)

Good practices are:

- Keep small number of rules by narrowing down the context
- Follow domain specific approach to build limited rule engine
- Test with production data & rule changes

Those points align well with microservice & DDD principles. The remaining part of this blog post demonstrates how to build a micro rule-engine by keep it small within a domain. Tools to be used are:

- Spring Boot & Spring MVC for REST
- Open API v3 for API contracts
- Drools as Rule Engine
- Kotlin
- Gradle and Jib to orchestrate build and containerisation

### Rules for validating election vote

In the election this year, I opted to vote by postal service. After the enrolment, AEC mailed me two ballot papers. The House of Representatives ballot has a list of candidates and you are asked to vote from 1 to n for all. The Senate ballot requires you to vote either above (parties) or below the black line (candidates + parties). There are rules to determine if the vote is valid:

1. You must be on electoral rolls 
2. You must be enrolled in postal vote 
3. Your security question must be answered and matched
4. Valid green ballot paper
5. Valid white ballot paper
6. Witnessed signature

### Define a rule fact model and an API contract

To start with, a rule fact model needs to be defined for the rule engine; the fact model should capture all information about the vote application (the fact). Once passed to the rule engine, it is executed against pre-defined rules & algorithms (e.g. Drools’ Rete, ReteOO or Phreak) and output the result. The rule engine takes care of processing and managing execution flow, stages and events.

The fact model can overlap with application domain model technically. For authorising the model, Drools provide a state of art Rule workbench. You can play with the workbench by `docker pull jboss/drools-workbench-showcase:latest`. 

The schema also can be hand-coded as POJOs; this time I use Swagger's OAS V3 to define APIs and generate Java classes. There are number of advantages:

- It fits as rule fact model
- Can be used for JSON SerDer
- Writing DRL (rule file) is easier with auto complete supported by IntelliJ

The swagger file for the application can be found [here](https://github.com/thanhnamit/api-election-v1/blob/master/src/main/resources/api/api.yaml). To start the Spring Boot app, simply use [this](https://start.spring.io/). Several extra dependencies are required as highlighted in comments in `build.gradle.kts`.

```java
...
plugins {
  ...
  id("org.hidetake.swagger.generator") version "2.18.1" // ***SWAGGER GENERATOR PLUGIN***
  id("com.google.cloud.tools.jib") version "1.2.0" // ***BUILD DOCKER IMAGE PLUGIN***
  ...
}
...
dependencies {
  ...
  implementation("org.drools:drools-core:7.19.0.Final") // ***DROOLS CORE***
  implementation("org.kie:kie-spring:7.19.0.Final") // ***SPRING SUPPORT***

  // ***SWAGGER GEN***
  implementation("io.swagger:swagger-annotations:1.5.22")
  swaggerCodegen("io.swagger.codegen.v3:swagger-codegen-cli:3.0.8")
  swaggerUI("org.webjars:swagger-ui:3.10.0")
  ...
}

...

// ***CONFIG GENERATION TASK***
val basePkg = "my.gov.election"
val apiSpec = "src/main/resources/api/api.yaml"
val srcDir = "src/main/java/my/gov/election"

swaggerSources {
  create("api") {
    code(closureOf<GenerateSwaggerCode> {
      inputFile = file(apiSpec)
      language = "spring"
      additionalProperties = mapOf(
          "modelPackage" to "$basePkg.models",
          "apiPackage" to "$basePkg.apis",
          "java8" to "true",
          "dateLibrary" to "java8",
          "interfaceOnly" to "true",
          "delegatePattern" to "false",
          "useTags" to "true"
      )
      dependsOn(validation)
    })
  }
}

tasks.withType<KotlinCompile> {
  kotlinOptions {
    freeCompilerArgs = listOf("-Xjsr305=strict")
    jvmTarget = "1.8"
  }
  // ***GENERATE CODE FIRST THEN COMPILE KOTLIN***
  dependsOn(swaggerSources["api"].code)
}

// TELL GRADLE WHERE TO FIND GENERATED CODE
sourceSets {
  main {
    java.srcDir(
      "${swaggerSources["api"].code.outputDir}/src/main/java")
  }
}

// TAG IMAGE WITH JIB
jib {
  to {
    image = "api-election-v1:$version"
  }
}
```
<pre></pre>

- To generate classes, simply run: `gradle generateSwaggerCodeApi`

Now the main part, we define a rule file `*.drl` to capture some validation rules. Drools also support other formats such as excel table, decision table, decision tree and they are more business friendly.

`vote-eligibility.drl`
```java
import my.gov.election.models.Vote
import my.gov.election.models.VoteEligibility
import my.gov.election.models.Ballot
import my.gov.election.models.Division
import my.gov.election.models.HouseOfRepBallot
import my.gov.election.models.SenateBallot
import my.gov.election.models.VotedPartyCandidates
import my.gov.election.models.VotedIndividualCandidates

global my.gov.election.models.VoteEligibility eligibility

dialect "java"

rule "Valid Vote 1 - By Party"
    when
        ob: Ballot()
        Vote(b: ballot != null)
        Vote(b.enrolmentId != null)
        Vote(b.division.id == ob.division.id)
        Vote(b.houseOfRepBallot.votedCandidates.size() == 3)
        Vote(b.senateBallot.voteType == SenateBallot.VoteTypeEnum.PARTY)
        Vote(b.senateBallot.votedPartyCandidates.candidates.size() == 1)
    then
        eligibility.setStatus(true);
        eligibility.setReason("Valid Vote 1");
end

rule "Valid Vote 2 - By Individuals"
    when
        ob: Ballot()
        Vote(b: ballot != null)
        Vote(b.enrolmentId != null)
        Vote(b.division.id == ob.division.id)
        Vote(b.houseOfRepBallot.votedCandidates.size() == 3)
        Vote(b.senateBallot.voteType == SenateBallot.VoteTypeEnum.INDIVIDUAL)
        Vote(b.senateBallot.votedIndividualCandidates.candidates.size() == 1)
    then
        eligibility.setStatus(true);
        eligibility.setReason("Valid Vote 2");
end
```

Next, define Drools beans and wire it to a Service class, which will be invoked by a Rest controller.

`src/main/kotlin/my/gov/election/Application.kt`
```java
@SpringBootApplication
class Application {
  @Bean
  fun kieContainer(): KieContainer {
    val svc = KieServices.Factory.get()
    val fs = svc.newKieFileSystem()
    fs.write(ResourceFactory.newClassPathResource("rules/vote-eligibility.drl"))
    svc.newKieBuilder(fs).buildAll()
    return svc.newKieContainer(svc.newKieBuilder(fs).kieModule.releaseId)
  }
}
```
<pre></pre>

`src/main/kotlin/my/gov/election/services/RuleEngine.kt`
```java
@Service
class RuleEngine(val kieContainer: KieContainer) {
    fun submitVote(vote: Vote): Vote {
        return vote.eligibility(ruleEngine.checkVote(vote, getBallot(vote.ballot.division.suburbs[0].postcode)))
    }
}
```
<pre></pre>

`src/main/kotlin/my/gov/election/services/VoteService.kt`
```java
@Service
class VoteService(val ruleEngine: RuleEngine) {
    ...
    fun submitVote(vote: Vote): Vote {
        return vote.eligibility(ruleEngine.checkVote(vote, getBallot(vote.ballot.division.suburbs[0].postcode)))
    }
}
```

Finally containerise the app with [Google Container Jib](https://github.com/GoogleContainerTools/jib): `gradle jibDockerBuild`. Note this command will create image locally only, the created image is quite small - 255MB. To run the container:

`docker run --rm -it -m 512M -p 8085:8085 api-election-v1:0.0.1-SNAPSHOT`

Invoke the API with [sample command](https://github.com/thanhnamit/api-election-v1/blob/master/README.md).

### Conclusion

I feel the learning curve is steep to become productive with Drools. With a simple rule sets, a careful approach in coding, design and a robust BDD test suite could be more manageable. Still, Drools is a great piece of software for Expert systems, but it could be more suitable for `experts` in this domain and some special problems (e.g. building promotion or pricing scheme for ecommerce catalog). Furthermore, data for the fact model should be collected prior to sending it to the engine's memory, which would require to front this service with other microservices to pre-process the data, or alternatively have the rule engine to react with events published from database systems.

Happy breaking rules! :feet:

