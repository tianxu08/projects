
# GraphQL in HZ

Integrating GraphQL to HZ's tech stack is my first project after I joined HZ.

### Introduction
GraphQL(http://graphql.org) is a data aggregation framework open sourced by facebook in 2015.
It replaces traditional RESTFUL by letting the client to describe exactly what it wants, and how response should look like, based on pre-defined object schema.

Based on GraphQL, we provided an framework where it is super easy for graphql developers to deliver highly maintainabl ecode and apply advanced features sucn as bathing without much scaffolding. 

### Goal
1. A common thin layer for all frontends, web or native, to retrieve HZ objects
2. An asynchronous system that’s high performant and low footprint
3. A strong type schema system with clear documentation

### Architecture
The following shows the architecture diagram. Steps breakdown:
1. Schema is defined (at least with mock data). Schema pushed to runtime
2. FE developer use graphiQL tool to construct query, review query performance                   
3. Optionally, FE developer can upload query to QMS to reduce payload over network
4. Mobile client calls server (middleware) with the qid + query variables + more
5. Meanwhile, web browser calls Jukwaa server asking for a page
6. Jukwaa web passes graphQL request to middleware (no network hop)
7. *qid -> graphQL query lookup (related to step 3, optional)
8. Backend federations, fan-out.
9. For both mobile and web, graphQL response is returned.
10. Meanwhile, graphQL execution metrics is logged. This updates graphiQL performance.

![overview](assets/overview.png)

### Ecosystem

The ecosystem contains 
1. Node.js with GraphQL schema and resolver, which are organized in the graph.
2. Utility Framework, leveraging JS decorators pattern and inspired by other frameworks such as SpringBoot.
3. Atom plugin, which makes it ieasier to create GraphQL nodes and connections
4. Monitoring system based on TSDB and Bosun. 
5. Development tools for higher productivity.

![ecosystem](assets/eco-system.png)


### Highlight
1. GrapHZ is the first framework in industry with ease of use and as a thin and lightweight layer. 
2. GrapHZ is well tested in large scale of traffic and works well.
3. GrapHZ is one of the most important engineering achievement in HZ in 2017.

### Details
##### The basics
A node is a core concept in our framework.

Nodes comprise of a schema and resolver and are organized like nodes in the graph. A node is more like a domain name such as "Photo", so that anything that has to do with a "Photo" should go inside this node:
```
graph
|-- serviceRegistry.json
|-- User
|-- Project
|-- Photo
      |-- Photo.js
      |-- Photo.gql
      |-- Test
            |-- PhotoTest.js
```
Example of Photo.gql
```
type Photo {
       id: Int
       owner: User
       ...
}
extend type Query {
       getPhotosByIds(ids: [Int]): [Photo]
}
```
Photo.gql and Photo.js come together. The .js is the corresponding resolver implementation:

Example of Photo.js
```
import schema from '/Photo.gql';
class Photo extends GraphNode {

constructor() {
    super();
}
getSchema() {
   return schema;
}
getResolver(self) {
     return {
          Photo: {
              @connect("getUserById", (obj) => {return {id: obj.userId}})
              owner(obj, args, context, info) {}
          },
          Query {
              @batch("ids")
              @service("photos")
              getPhotosByIds(obj, args, context, info, service) {
                     const { ids } = args;
                    return service.photos.getSpacesById(ids)
                    .then((response) => {
                          var photos = getPhotosFromResponse(response, ids);
                          return photos;
                     });
            }
          }
     };
}
}
module.exports = new Photo();
```
A node always extends our GraphNode class which has certain methods to implement such as getSchema and getResolver. What is returned by getResolver directly maps to what is defined in the .gql file.

The usage
The framework automatically registers nodes into runtime and a graphql query can be issued against the graphql engine before or during page rendering phase:

var graphQLPromise = Graphouzz.getClient().query(cityProfileGql, variables)
.then(result => {
     ...
})
The annotations
The framework is built heavily leveraging JS decorators pattern and inspired by other frameworks such as SpringBoot. This is how cleanness is achieved.

@connect

Objects are usually inter-connected in graphQL. That's what we are seeing here between User and Photo: Photo will have an owner field whose type is User. The framework provides an easy way to connect the owner field to a resolver implementation that can fulfill it:

@connect("getUserById", (obj) => {return {id: obj.userId}})
owner(obj, args, context, info) {}
The annotation intercepts resolver and returns a promise of User type from getUserById resolver, which is another top level query field. This way fields can be connected at will.

@batch

Consider the following resolver implementation:

Project

@connect('getPhotosByIds', (obj) => {return [ids: [obj.photoId]}}
Photo(...)
When there's a list of 10 projects each with a different photo, then the photo backend will be called 10 times. This is known as N+1 problem.

To solve the N+1 problem, batching is supported. Different from other approach such as graphql-batch, batching here happens before resolver, instead of inside the resolver. This means batching is nothing but a flood gate to the resolver: it tries to aggregate as much attempted calls to resolver as possible and then flush out with a single call to it:

@batch("ids")
getPhotosByIds(obj, args, context, info, service) {
     const { ids } = args;
     ...                   
}
The annotation declares that the batching is based on the ids field. This ids field should have already been defined as a plural param in schema. When the getPhotosByIds is called via @connect, the call is intercepted by the batch annotation and a promise immediately returned. It's guaranteed that there will only be a single execution of resolver code within a JS tick.

@service

Nothing is real without a concrete call to backend. Backend instance can be created with the service annotation. You can declare as many different service instances as you want and they will all be injected into the service obj that's passed along into the resolver. This is an extension to existing resolver interface:

@service('aBackend')
@service('anotherBackend')
resolver(...service) {
      service.aBackend(...)
      service.anotherBackend(...)
      ...
}
There's a top level serviceRegistry.json config that defines how these named services should be mapped to real code. This is merely a simple import mapping.

There's a special thrift wrapper service that you can import instead so that minimum client-side code needs to be written to call the thrift backend. The wrapper service looks into your generated thrift folder and know how to call each thrift endpoints. The wrapper also make it possible to instrument and monitor service latency etc.

@plural

We believe that the schema should be as complete as possible. That's why we make it easy to create singular form endpoint out of plural form:

@batch("ids")
getPhotosByIds(...) 
@plural('getPhotosByIds', (obj, args) => {return {ids: [args.id]};})
getPhotoById(...)
Plural annotation is a special form of connection that connects the singular resolver to its plural form. You only need to tell the annotation how to transform from singular to plural (e.g. id to ids mapping), and there's no need to implement anything inside the singular resolver because of the plural connection.

This also make it easier when building connection as the connection is usually in singular form.

The Atom plugin
All 4 kinds of annotations above can be easily created via a custom atom plugin.

(https://drive.google.com/open?id=1ccYNglFxkFOKHMOttAE8T2N_MqT-m9iX)

For service annotation, the plugin will auto suggest the available services defined in the serviceRegistry.
For connect annotation, the plugin will also auto suggest a list of available connections.

The code-gen

The skeleton code of Photo.js can actually be auto-generated, via a custom Yeoman scaffolding robot. This further reduces the amount of code a developer has to write. In fact, the script generates everything inside Photo folder, including a dummy Mocha test:

(https://drive.google.com/open?id=1mzouVgi3IfUnOcVBB63aOPEQrs92rtXn)

The testing harness

We have built an Jest-inspired, extensible testing infra under Mocha context to make it easy to write stable unit tests. The tests are mainly to ensure the query and response align with each other.

Simple 1-1 diff-based test can be achieved directly using the GraphiQL test autogen. But in real world data can change over time (e.g. timestamp field) so we have added helper test functions that tell the test infra what is considered a "pass".

Some helper test functions we have built:

exist: test if the field value is null or not. As long as it's not null, it's a pass.
isArray: test if the field value is a JSON array. As long as it's an array, it's a pass.
arrayContains: test if the JSON array contains certain list of objects.
like: test if the field value (sub object) look alike what we are expecting. As long as certain threshold percentages of values are matched, it's a pass.
The testing harness is extensible as more custom helpers functions can be added.
