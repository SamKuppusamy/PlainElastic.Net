PlainElastic.Net
=======

The really plain Elastic Search .Net client.

### Plain Idea 

Usually connectivity clients built using **BLACK BOX** principle: **there is a client interface and some unknown magic behind it**.<br/>
*(call of the client method internally generate some commands and queries to external system, get responses, somehow process them and then retrieve result to user)*<br/>
As the result user hardly can debug connectivity issues or extend client functional with missed features.

The main Idea of PlainElastic.Net is to be a **GLASS BOX**. e.g. provide a **full control over connectivity process to user**.


### How Its works

1) The only thing you need to connect to ES is a HTTP connection.

```csharp
  var connection  = new ElasticConnection();
```

2) Than you can write stings with ES command 
  
```csharp
  string command = "http://localhost:9200/twitter/user/test";
```

3) And JSON string with data

```csharp
	string jsonData = "{ \"name\": \"Some Name\" }";
```

4) And pass them using connection to ES.

```csharp
	string response = connection.Put(command, jsonData);
```

5) Get JSON string response and analyze it.

```csharp	
  if(response.Contains("\"ok\":true")) {
	 ... // do something useful
	}
```

#### So, how PlainElastic can help you here?

```csharp
  // 1. It provides ES HTTP connection
  var connection  = new ElasticConnection("localhost", 9200);
  
  // 2. And sophisticated ES command builders:
  string command = Commands.Index(index: "twitter", type: "user", id: test)
  
  // 3. And gives you the ability to serialize your objects to JSON:  
  var serializer = new JsonNetSerializer();
  var tweet = new Tweet { Name = "Some Name" };
  string jsonData = serializer.ToJson(tweet);
  
  // 4. Then you can use appropriate HTTP verb to execute ES command:
  string response = connection.Put(command, jsonData);
  
  // 5. And then you can deserialize operation response to typed object to easily analyze it:
  IndexResult indexResult = serializer.ToIndexResult(result);
  if(indexResult.ok) {
     ... // do something useful.
  }
  
  // 6. And even more: Typed mapping and condition-less query builders.
```

### Concepts

#### No addition abstraction upon native Elastic Search query and mapping syntax.

This eliminates requirements to read both ES and driver's manuals, 
and also it allows you not to guess how driver will generate actual ES query when you construct it using driver's Query DSL.<br/>
*So if you want to apply some ES query - all you need is to read [ES Query DSL documentation](http://www.elasticsearch.org/guide/reference/query-dsl/)*


#### All you need is strings.

Let's take some ES query sample in a format that you will see in ES documentation:
	
```
$ curl -XGET http://localhost:9200/twitter/tweet/_search -d '{
   	"query" : {
       	"term" : { "User": "somebody" }
   	}
}'
```

In PlainElastic.Net this could be done using:

```csharp
var connection  = new ElasticConnection("localhost", 9200);
string command = new SearchCommand("twitter", "tweet"); // This will generate: twitter/tweet/_search
string query = new QueryBuilder<Tweet>()				// This will generate: 
					.Query(q => q						// { "query": { "term": { "User": "somebody" } } }
						.Term(t => t
							.Field(tweet=> tweet.User).Value("somebody")
						)
					).Build()
string result = connection.Get( command, query);

// Than we can convert search results to typed results
var serializer = new JsonNetSerializer();
var foundTweets = serializer.ToSearchResults<Tweet>(result);
foreach (Tweet tweet in  foundTweets.Documents)
{
	...
}
```

As you can see *all parameters* passed to and returned from Get HTTP verb execution are just **strings**.<br/> 
This give us complete control over generated commands and queries. You can copy/paste and debug them in any ES tool that allows to execute JSON queries (e.g. CURL or ElasticHead ).
	

### Command building

PlainElastic.Net commands represent URL part of ElasticSearch requests.<br/>
All commands have corresponding links to ES documentation in their XML comments, 
so you can use these links to access detailed command description.

Most of the commands have *Index* ,*Type* and *Id* constructor parameters, *(these parameters forms address part)*
all other options could be set using fluent builder interface.

```csharp
string indexCommand = new IndexCommand(index: "twitter", type: "tweet", id: "10")
						 	.Routing("route_value")
						 	.Refresh();
```

There is also a Commands class that represents a command registry and allows you to easily build commands,
without necessity to remember command class name.

```csharp
string searchCommand = Commands.Index(index: "twitter", type: "tweet", id: "10")
						 	.Routing("route_value")
						 	.Refresh();
```

### Indexing
	
*ES documentation:*  http://www.elasticsearch.org/guide/reference/api/index_.html

The easiest way to index document is to serialize your document object to JSON and pass it to PUT index command:

```csharp
var connection  = new ElasticConnection("localhost", 9200);
var serializer = new JsonNetSerializer();

var tweet = new Tweet { User = "testUser" };
string tweetJson = serializer.ToJson(tweet);

string result = connection.Put(new IndexCommand("twitter", "tweet", id: "10"), tweetJson);

// Convert result to typed index result object. 
var indexResult = serializer.ToIndexResult(result);
```

**Note:** You can specify additional indexing parameters such as Parent or Refresh in IndexCommand builder.

```csharp
string indexCommand = new IndexCommand("twitter", "tweet", id: "10").Parent("5").Refresh();
```

### Bulk Operations

*ES documentation:*  http://www.elasticsearch.org/guide/reference/api/bulk.html

There are two options to build Bulk operations JSONs.
First is to build all Bulk operations at once:

```csharp
IEnumerable<Tweet> tweets = new List<Tweet>();

string bulkCommand = new BulkCommand(index: "twitter", type: "tweet");

string bulkJson = 
    new BulkBuilder(serializer)
       .BuildCollection(tweets,
            (builder, tweet) => builder.Index(data: tweet,  id: tweet.Id)
                       // You can apply any custom logic here
                       // to generate Indexes, Creates or Deletes.
);

string result = connection.Post(bulkCommand, bulkJson);

//Parse bulk result;
BulkResult bulkResult = serializer.ToBulkResult(result);
...
```

Second allows you to build Bulk operations in batches of desired size.<br/> 
This will prevent from constructing huge in-memory strings, and allows to process input collection on-the-fly,
without enumerating them to the end.

```csharp
IEnumerable<Tweet> tweets = new List<Tweet>();

string bulkCommand = new BulkCommand(index: "twitter", type: "tweet");

IEnumerable<string> bulkJsons = 
    new BulkBuilder(serializer)
        .PipelineCollection(tweets,
            (builder, tweet) => builder.Index(data: tweet,  id: myObject.Id))
        .JoinInBatches(batchSize: 10); // returns deferred IEnumerable of JSONs  
                            // with at most 10 bulk operations in each element,
                            // this will allow to process input elements on-the-fly
                            // and not to generate all bulk JSON at once

foreach(string bulk in bulkJsons )
{
  // Send bulk batch.
  string result = connection.Post(bulkCommand, bulk);

  // Parse bulk batch result.
  BulkResult bulkResult = serializer.ToBulkResult(result);
  ...
}
```

**Note:** You can build not only *Index* Bulk operations but also *Create* and *Delete*.
	
```csharp
IEnumerable<string> bulkJsons = 		
	new BulkBuilder(serializer)
		 .PipelineCollection(tweets,
            (builder, tweet) => {
            	switch (tweet.State) {
            		case State.Added: 
            			builder.Create(data: tweet,  id: myObject.Id))
            		case State.Updated: 
            			builder.Index(data: tweet,  id: myObject.Id))
            		case State.Deleted:
            			builder.Delete(id: myObject.Id))
            	}
            });
```

### Queries
*ES documentation:*  http://www.elasticsearch.org/guide/reference/query-dsl/

The main idea of QueryBuilder is to repeat JSON syntaxes of ES queries.<br/>
Besides this it provides *intellisense* with fluent builder interface <br/>
and *property references*:

for single property `.Field(tweet => tweet.Name)` <br/>
for collection type property `.FieldOfCollection(collection: user => user.Tweets, field: tweet => tweet.Name)`


So let’s see how it works.

We have *http://localhost:9200/twitter* index with type *user*.
Below we add sample "user" document to it:

```json
PUT http://localhost:9200/twitter/user/1
{
    "Id": 1,
    "Active": true,
    "Name": "John Smith",
    "Alias": "Johnnie"
}
```

Now let's create some synthetic JSON query to get this document:

```json
POST http://localhost:9200/twitter/user/_search
{
    "query": {
        "bool": {
            "must": [
                {
                   "query_string": {
                      "fields": ["Name","Alias"], "query" : "John" 
                    }
                },
                {
                   "prefix" : {
                      "Alias": { "prefix": "john" } 
                   }
                }
            ]
        }
    },
    "filter": {
        "term": { "Active": "true" }
    }
}
```

Assuming that we have defined class User:

```csharp
class User
{
    public int Id { get; set; }
    public bool Active { get; set; }
    public string Name { get; set; }
    public string Alias { get; set; }
}
```

This query could be constructed using:

```csharp
string query = new QueryBuilder<User>()
    .Query(q => q
        .Bool(b => b
           .Must(m => m
               .QueryString(qs => qs
                   .Fields(user => user.Name, user => user.Alias).Query("John")
               )
               .Prefix(p => p
                    .Field(user => user.Alias).Prefix("john")
               )
           )
        )
    )
    .Filter(f => f
        .Term(t => t 
            .Field(user=> user.Active).Value("true")
        )
    )
    .BuildBeautified();
```

And then to execute this query we can use the following code:
```csharp
var connection = new ElasticConnection("localhost", 9200);
var serializer = new JsonNetSerializer();

string result = connection.Post(Commands.Search("twitter", "user"), query);
User foundUser = serializer.ToSearchResult<User>(result).Documents.First();
```

See [Query Builder Gist](https://gist.github.com/2765230) for complete sample.


#### Condition-less Queries:

Its usual case when you have a bunch of UI filters to define full-text query, price range filter, category filter etc.<br/>
None of these filters are mandatory, so when you construct final query you should use only defined filters.
This brings ugly conditional logic to your query-building code.

So how PlainElastic.Net addresses this?

The idea behind is really simple:<br/> 
**if provided query or filter value is null or empty - the whole query or filter will not be generated.**

Expression 

```csharp
string query = new QueryBuilder<User>()
    .Query(q => q
        .QueryString(qs => qs
           .Fields(user => user.Name, user => user.Alias).Query("")
        )
    )
    .Filter(f => f
        .Term(t => t 
            .Field(user=> user.Active).Value(null)
        )
    )
    .Build();
```

will generate "{}" string that will return all documents from the index.

The real life usage sample: <br/>
Let's say we have criterion object that represents UI filters:

```csharp
class Criterion
{
    public string FullText { get; set; }
    public double? MinPrice { get; set; }
    public double? MaxPrice { get; set; }
    public bool? Active { get; set; }
}
```

So our query builder could look like this: 

```csharp
public string BuildQuery(Criterion criterion)
{
    string query = new QueryBuilder<Item>()
        .Query(q => q
            .QueryString(qs => qs
                .Fields(item => item.Name, item => item.Description)
                .Query(criterion.FullText)
            )
        )
        .Filter(f => f
            .And(a => a
                .Range(r => r
                    .Field(item => item.Price)                           
                    // AsString extension allows to convert nullable values to string or null
                    .From(criterion.MinPrice.AsString())
                    .To(criterion.MaxPrice.AsString())
                )
                .Term(t => t
                    .Field(user => user.Active).Value(criterion.Active.AsString())
                )
            )
        ).BuildBeautified();
}
```

And that's all: no ugly ifs or switches.<br/>
You just write query builder using most complex scenario, and then it will build only defined criterions.

If we call this function with `BuildQuery( new Criterion { FullText = "text" })`
then it will generate:

```json
{
    "query": {
        "query_string": {
            "fields": ["Name", "Description"],
            "query": "text"
        }
    }
}
```

so it will omit all not defined filters.

See [Condion-less Query Builder Gist](https://gist.github.com/2765335) for complete sample.

### License

PlainElastic.Net is free software distributed under the terms of MIT License (see LICENSE.txt) these terms don’t apply to other 3rd party tools, utilities or code which may be used to develop this application.

