## NLP endpoints: Haystack plus FastAPI

In this article, we will build a **FastAPI application** which interacts with **haystack**. It's an experiment intended to be a starting point for anyone wishing to use haystack as a service for other projects. And one of the easiest ways is through a REST API.
You can deploy a REST API into a physical machine, a virtual machine, a container, or a serverless environment (like AWS Lambda or Azure Functions).

Let's code.

## The virtual environment

Everybody loves Python! 

Who doesn't get amazed to "talk with the code"? When coding in Python in particular, I feel like I'm talking to my computer. For someone coding since when I was 12yo, you may understand my passion. 

Python has thousands of available modules, and each day more become available. When you need something, look for it on http://pypi.org/. It's hard not to find something useful there! But this reminds me of one of the most famous sentences in a movie: *"With great power comes great responsibility."* If we keep installing modules into a mixed environment, we are laying the groundwork for considerable future problems in the dependency chain, testing, and organization. The recommended way is to use **isolated environments**, which can be achieved by using [python-venv](https://docs.python.org/3/library/venv.html). Note, you might consider the use of dependency managers such as [Poetry](https://python-poetry.org/) or [PDM](https://pdm.fming.dev/) - there are plenty of articles around the web that describe why and how you might use them. 

### Create a new environment.

```python
python3 -m venv venv
``` 




### Activate the virtual environment.

```python
source venv/bin/activate
``` 


### Install haystack

Now we install the outstanding **[haystack](https://haystack.deepset.ai)** NLP framework. 

We are using [OpenSearch](https://opensearch.org/) as our Document Store. For this, we set the egg *opensearch* to instruct pip to install its related dependencies together with the framework. You can find the appropriate commands for using different Document Stores [here](https://github.com/deepset-ai/haystack/blob/e1f399284fad6a5f04d7d95bcccc3056ddac03b1/pyproject.toml#L97). If you just want to use Elasticsearch, then remove the egg altogether.


```
pip install fastapi
```


### Create an empty python file

```python
touch app.py
```

Here is the https://github.com/danielbichuetti/haystack_fastapi_endpoint/blob/main/app.py of this file for your reference. But it would be very beneficial for you to follow along and build the application step-by-step.

### The health-check endpoint

First, it is good practice to expose a health-check endpoint. But what is a health-check endpoint?

It's a route that tells external systems the current status of your application. For example, normal states are running correctly (green), partially (yellow), or not running correctly (red). Some providers can check this endpoint to know if your app is running correctly and doesn't need a restart. It's also useful by allowing you to set up alarms in monitoring tools.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}
```

This code isn't doing extended checks and returning possible warnings (yellow reasons) or errors (red reasons). But a production API really should. It's a starting point, and what it should return when not working should be evaluated on a case-by-case basis for each application and business.

### Async capabilities

Besides offer the typical synchronous connection methods, FastAPI also supports asynchronous methods. This allows the application to carry-on with other tasks in its queue while it waits for an external process to complete. In IO-heavy applications, asynchronous code is critical to avoid bottlenecks and improve performance. Therefore, you need to add **async** into your method definition.


```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "ok"}
```

### Running your FastAPI application

Ok, now you may be thinking about how to actually run this application. Perhaps you are considering calling "python  app.py"? But, no, FastAPI is a framework, not a server. Therefore, you must use something else. Here is where [uvicorn](https://www.uvicorn.org/) is helpful. There is also [gunicorn](https://gunicorn.org/), a WSGY server, but this is for another article. 

Install it.

```
pip install uvicorn[standard]
```


From the command line, you can now run:

```
uvicorn app:app --reload
```

The first *app* is the python filename without the extension, and the second is the FastAPI *App object*. The reload option will make uvicorn reload the file whenever it's changed. Only use this while developing in order to have the server automatically reload when you make changes to the app.py file.


It will start on the default port, 8000. So you should see something like this on the command line:

```
INFO:     Will watch for changes in these directories: ['/home/danielbichuetti/Dev/Projects/fastapi_haystack']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [57303] using WatchFiles
INFO:     Started server process [57307]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

### Testing the skeleton application

```curl
curl http://localhost:8000/health
```

The response should be:

```
{"status":"ok"}
```

Yeah, it's a JSON string. Pretty easy to build a REST API, right? Its very straightforward and fast to configure, because FastAPI handles the rest.

## Deploy an OpenSearch cluster

Do you have any OpenSearch servers available? If yes, you can jump to the next section. If not, you will love this one.

### Create an AWS OpenSearch Service

Go to [AWS console](https://signin.aws.amazon.com/signin) portal and log in. If you are not a user yet, please [sign up](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html?refid=c623d581-46f6-43a2-b227-cabbee9cd673) for a free account. 

AWS provides Amazon OpenSearch Service on their free tier. So yeah, 750 hours of **t3.small.search** instance monthly. It's a good starting point.

After you have logged in, find [Amazon OpenSearch Service](https://us-east-1.console.aws.amazon.com/opensearch). Then click on *Create Domain*

![Screenshot from 2022-09-04 15-06-45.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662314815773/Wk9mwnVpp.png align="left")

You will get redirected to the cluster creation screen. The first field is the name you want to use for your cluster. Let's call it **haystack**.


![Screenshot from 2022-09-04 15-09-30.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662314979663/d_GLHt_LS.png align="left")

Choose the Deployment type, Development and testing, and select 1.3 (latest) as the version. 


![Screenshot from 2022-09-04 15-11-23.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315092919/RNYk4t9Lk.png align="left")

Let's tweak some default Data nodes options to keep setup simple and use just the free tier.
First, let's use only **1 AZ**, and change the **Number of nodes** to **1**. On **Instance Type**, change it to **t3.small.search**. Don't change the EBS size to more than 20GB, as it's the maximum allowed for free.


![Screenshot from 2022-09-04 15-17-10.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315439686/AuhiOOKJ7.png align="left")

Scroll down until you get to the **Network** part. Usually, this is a bad idea without further tuning and control. However, for this experiment, it will be ok. We will use **Public access** for the network, which means *we exposed our cluster endpoints to the Internet*.


![Screenshot from 2022-09-04 15-18-47.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315533432/rflTcTdTA.png align="left")

Because we exposed OpenSearch to the Internet, set up at least an **admin user**. I would recommend using IAM roles or certificates in a production environment. Let's call the user "hs_admin" and set up a password.


![Screenshot from 2022-09-04 15-23-42.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315830661/JaXxm7R6J.png align="left")

For a more straightforward setup and to avoid extensive configuration, let's **turn off domain access policies** and just use fine-grained access control. **Avoid this on production**. 


![Screenshot from 2022-09-04 15-25-22.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315939281/iS-nTTYGa.png align="left")


Now hit the Create button on the bottom of the page and wait.

You'll see a page with loading information, like this one. It shows the steps of the cluster creation process. You will have to wait a few minutes.


![Screenshot from 2022-09-04 15-27-09.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662316104372/cewhvyfHo.png align="left")


After some time, your cluster status will turn **Green**, like this:


![Screenshot from 2022-09-04 15-47-23.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662317291298/Mbwf1q0Gw.png align="left")

Click on your recently created cluster: haystack.

You will see something like this:


![Screenshot from 2022-09-04 15-47-23.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662317342180/s2K42ZA25.png align="left")

The information that is important for us is the **Domain endpoint**, which is your cluster address:


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662317412430/Yq2nenEjj.png align="left")

Here we end up with the creation of OpenSearch. Back to the code (finally).

## Refactoring the application

### The environment values

We must provide the authentication data to connect to our OpenSearch cluster. Putting this directly in code is a terrible decision. Never do it. It would be best if you used secret stores or environment variables.

We will now create a *.env* file in our project directory. You will fill it with the cluster address, the admin username, and the password. Remember to paste the server address without the scheme (HTTPS or HTTP), as well as the port. These are extra parameters for haystack OpenSearchDocumentStore.


```
OPENSEARCH_SERVER=search-haystack-4luhmrfigopcmjfnvh62rtsp5q.us-east-1.es.amazonaws.com
OPENSEARCH_SERVER_PORT=443
OPENSEARCH_USER=hs_admin  
OPENSEARCH_PASSWORD=H2YsT2CkPa$$w0rd
```

We will install a library to help load environment variables during development.

```
pip install python-dotenv
```

In your app.py file, add. 

```
import os
from dotenv import load_dotenv

load_dotenv()
OPENSEARCH_SERVER = os.getenv("OPENSEARCH_SERVER")
OPENSEARCH_SERVER_PORT = os.getenv("OPENSEARCH_SERVER_PORT")
OPENSEARCH_USER = os.getenv("OPENSEARCH_USER")
OPENSEARCH_PASSWORD = os.getenv("OPENSEARCH_PASSWORD")
```

The variables from the environment will get loaded, but before that, dotenv will check whether they are present or not. If not, it will use the .env data.



### Initialize OpenSearchDocumentStore

With this in hand, we can now initialize our haystack DocumentStore:

```
document_store = OpenSearchDocumentStore(
    host=OPENSEARCH_SERVER,
    port=OPENSEARCH_SERVER_PORT,
    username=OPENSEARCH_USER,
    password=OPENSEARCH_PASSWORD,
    verify_certs=True
)
```

There are other parameters and ways of fine-tuning this initialization; you can read more [here](https://haystack.deepset.ai/reference/document-store#opensearchdocumentstore).


### Data validation

Valid data in a REST endpoint is a need. We don't want anyone sending invalid data to our endpoints. Therefore, we will use data validation.

Let's define a model for the requests made to our endpoints. First, we will send documents, which will be JSON documents with content and a name.

We will use [pydantic](https://pydantic-docs.helpmanual.io/) for it. First, we will install it:

```
pip install pydantic
```

Then we will declare the new class:

```
from pydantic import BaseModel
from typing import Optional

class Document(BaseModel):
    name: Optional[str] = None
    content: str
```

Here there is an important point to take note of. Take care with naming classes on your project to avoid overwriting other classes. You can see I used Document. But then I remembered one of the most critical [primitives in haystack](https://haystack.deepset.ai/reference/primitives): Document. Like Labels and Answers, it's a pillar of it. So we will use an alias on the import like this: 

```
from haystack.schema import Document as HaystackDocument
```

Let's keep the alias to avoid overwriting Document while still getting in touch with the best NLP framework.

### Add a document

We will define one way to add a document to our Document Store.

```python
@app.post("/documents", status_code=201)
def save_document(document: Document):
    new_doc= HaystackDocument(content=document.content, meta={"name": document.name})
    document_store.write_documents([new_doc])
    return {"id": new_doc.id}
```

We test the endpoint:

```curl
curl -X POST http://localhost:8000/documents/ -H 'Content-Type: application/json' -d '{"name":"My first haystack document", "content":"Haystack is an open-source framework for building search systems that work intelligently over large document collections."}'
```

We didn't get a response!

Let's take a look at the Uvicorn log that should be running in your console:

`INFO:     127.0.0.1:35134 - "POST /documents/ HTTP/1.1" 307 Temporary Redirect`

It seems that it is connecting, however the redirect doesn't seem correct.

Lets check the syntax. FastAPI has a built-in mechanism called Swagger and you can check and test the syntax for all of your defined endpoints if you visit http://localhost:8000

![msedge_8R1RzsEXeg](https://user-images.githubusercontent.com/88559987/188543512-44412dcf-cdbc-40ca-a084-e5c400475b31.gif)

This generated the following curl request: 

```
curl -X 'POST' \
  'http://localhost:8000/documents' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "string",
  "content": "string"
}'
```

If we inspect closely, we can see that the url differs *slightly* from the one we used --- there isn't any trailing slash after `documents`

Typically a trailing slash is used to denote a resource group, while a URL without a slash names a specific resource. Since we are targeting a specific endpoint, let's remove that slash and try again.

```curl
curl -X 'POST' 'http://localhost:8000/documents' -H 'Content-Type: application/json' -d '{"name":"My first haystack document", "content":"Haystack is an open-source framework for building search systems that work intelligently over large document collections."}'
```

Voila! That was easy and fast! You should see a response like this:

```
{"id":"e41e6aeb0ae5965a912b672664e58b7c"}
```

It's the id that haystack generated for your document. Let's copy that somewhere for future use.


### Get a document

What if we want to retrieve this document from the datastore?

Let's build an endpoint to read a document by its id:

```python
from fastapi import Response

@app.get("/documents/{document_id}", status_code=200)
def get_document(document_id: str, response: Response):
    document = document_store.get_document_by_id(document_id)
    if document is None:
        response.status_code = 404
        return {"error": "Document not found"}
    return document_store.get_document_by_id(document_id)
```

We are trying to get the document from haystack OpenSearchDocumentStore. If there isn't a document with this id, you will receive a 404 HTTP response code along with an error message. On the other hand, if the document exists, you will receive a 200 HTTP response code with the document.

```curl
curl http://localhost:8000/documents/e41e6aeb0ae5965a912b672664e58b7c
```

Our response will have a bit more information than what we first sent to create the endpoint:

```
{"content":"Haystack is an open-source framework for building search systems that work intelligently over large document collections.","content_type":"text","id":"e41e6aeb0ae5965a912b672664e58b7c","meta":"name":"My first haystack document"},"score":0.5312093733737563,"embedding":null}
```

You can learn more about haystack's Document primitive [here](https://haystack.deepset.ai/reference/primitives#document).

By the way, the haystack documentation is very comprehensive - it not only covers the technical details of the API, but also offers a lot of background information, Tutorials and other Guides. Spend some time there -- you will quickly learn about the immense possibilities that Haystack opens you up to!


### Add more documents

```curl
curl -X POST http://localhost:8000/documents -H 'Content-Type: application/json' -d '{"name":"deepset Cloud", "content":"Build production-ready NLP services.  deepset Cloud is a SaaS platform to build natural language processing applications."}'
```

```curl
curl -X POST http://localhost:8000/documents -H 'Content-Type: application/json' -d '{"name":"FastAPI", "content":"FastAPI is a modern, fast (high-performance), web framework for building APIs with Python 3.6+ based on standard Python type hints."}'
```

### Get all documents

Now, what if we want to get all documents? Recall what we said about resource groups vs specific resources? Since we're trying to retrieve an entire group, let's use a trailing slash this time.

 ```python
@app.get("/documents/", status_code=200)
def get_all_document(response: Response):
    documents = document_store.get_all_documents()
    if documents is None:
        response.status_code = 404
        return {"error": "Documents not found"}
    return documents
```

We can test this endpoint with the following:

```curl
curl http://localhost:8000/documents/
```

This time our response will come with all three of the documents that we have added.

```
[{"content":"FastAPI is a modern, fast (high-performance), web framework for building APIs with Python 3.6+ based on standard Python type hints.","content_type":"text","id":"5209b538869b938ac94bf70fa0b09bd","meta":{"name":"FastAPI"},"score":null,"embedding":null},{"content":"Build production-ready NLP services.  deepset Cloud is a SaaS platform to build natural language processing applications.","content_type":"text","id":"4985f3369c9c9c6a6ba179a004b0af72","meta":{"name":"deepset Cloud"},"score":null,"embedding":null},{"content":"Haystack is an open-source framework for building search systems that work intelligently over large document collections.","content_type":"text","id":"e41e6aeb0ae5965a912b672664e58b7c","meta":{"name":"My first haystack document"},"score":null,"embedding":null}]
```

There is one crucial thing that you must always pay attention to. **Never**, never in any production return all entities into a single call, be it on HTTP, GRPC, SQL query. Please, never do this! It's a horrible practice. We only permitted this here because we knew that only three documents existed in the index. But imagine the damage you would cause to your servers if you had 10 million documents! 

Instead, you should use *pagination techniques*. Fortunately for us, haystack comes to the rescue here - the `get_all_documents()` function that we used has a default batch_size of 10000, meaning that that's the maximum amount of documents that will be returned if we don't otherwise set it. Likewise, there are [default batch_sizes set](https://github.com/deepset-ai/haystack/pull/2958/files) throughout the haystack code to prevent any catastrophes from happening.



***NOTE TO DANIEL - I tried using batch_size values such as 1 or 2, and it returned all of them. Why? Is there a bug in the mechanism?***


### Querying the Document Store directly

Before we move on to the real power of haystack --- its nodes and pipelines --- we will test out one final mechanism: running [custom DSL queries](https://haystack.deepset.ai/reference/document-store#baseelasticsearchdocumentstore) directly on our Document Store.

Let's set up a simple "search" endpoint. 

```
import logging

@app.get("/documents/search/{query}", status_code=200)
def search_document(query: str, response: Response):
    logging.debug(f"Searching for {query}")
    documents = document_store.query(query)
    return documents
```

We can call it by running:

```curl
curl http://localhost:8000/documents/search/saas
```

Our response will be:

```
[{"content":"Build production-ready NLP services.  deepset Cloud is a SaaS platform to build natural language processing applications.","content_type":"text","id":"4985f3369c9c9c6a6ba179a004b0af72","meta":{"name":"deepset Cloud"},"score":0.508989096965447,"embedding":null}]
```

You must keep two things in mind for this elementary search and query:

1. There are only three documents in our Document Store
2. SaaS keyword is present in only one document.
3. The query is concise and limited

If we had more documents with that search term, the BM25 algorithm would help us quickly retrieve and sort the matches by their relevance.

Haystack also has a powerful filtering mechanism, which would allow us to filter a more complex dataset across many dimensions:

```
filters = {
    "$and": {
        "type": {"$eq": "article"},
        "date": {"$gte": "2015-01-01", "$lt": "2021-01-01"},
        "rating": {"$gte": 3},
        "$or": {
            "genre": {"$in": ["economy", "politics"]},
            "publisher": {"$eq": "nytimes"}
        }
    }
}
```

Note: This filter doesn't apply to our current application, it's just an example to understand its logic.

If we were to be embedding this into an existing Elasticsearch application that generates DSL JSON queries, we could also pass that query along to Haystack, which would then pass it along to the Document Store to be processed normally. Please see the [documentation](https://haystack.deepset.ai/reference/document-store#baseelasticsearchdocumentstore) to learn more about the possibilities.

But haystack is much more powerful than this. So far we have only used a tiny piece of the framework - indexing and retrieving basic text documents. Now it is time to explore the outstanding power of haystack Nodes.

### Extractive answer endpoint

We start by adding some imports:

```
from haystack.nodes import BM25Retriever, FARMReader
from haystack.pipelines import ExtractiveQAPipeline
```

We define a pydantic model:

```
class Query(BaseModel):
    question: str
```

Then we add the endpoint:

```python
@app.post("/documents/ask", status_code=200)
def ask_document(query: Query, response: Response):   
    model = "deepset/roberta-base-squad2"
    retriever = BM25Retriever(document_store)
    reader = FARMReader(model, use_gpu=True)
    pipeline = ExtractiveQAPipeline(reader, retriever)
    result = pipeline.run(query=query.question, params={"Retriever": {"top_k": 10}, "Reader": {"top_k": 1}})    
    answers = [x.to_dict() for x in result["answers"]]
    return answers
```

Let's talk a bit about this endpoint. 

First, we are defining which model we will use. The company behind haystack, [deepset.ai](https://www.deepset.ai/), has publicly shared many astonishing models on [Hugging Face](https://huggingface.co/deepset), which is the de facto respository and community for sharing ML NLP models. You will find small, medium, and giant models there and all of them have excellent quality, so please explore them. Oh, and don't forget to give Deepset a like on Hugging Face! 

Then we set up a BM25Retriever, which will do a sparse search - using just simple text keywords - for us. 

After that, we set up the FARMReader (our friends at deepset also developed FARM, it's an excellent framework for transfer learning). 

We instantiate one of the [ready-made pipelines](https://haystack.deepset.ai/components/ready-made-pipelines). These pipelines fit the most common search patterns and chain the appropriate haystack components together automatically. For example, the [ExtractiveQAPipeline](https://haystack.deepset.ai/components/ready-made-pipelines#extractiveqapipeline) searches the OpenSearch document collection and extracts a piece of text (a span) that answers our question.

We call the endpoint:

```curl
curl -X POST http://localhost:8000/documents/ask -H 'Content-Type: application/json' -d '{"question":"What is deepset Cloud?"}'
```

When we access this endpoint for the first time, it will take a little while because haystack will need to download the model and then run it. The time will vary depending on the model (deepset/roberta-base-squad2 is about 480 MB) and your connection speed. But it's a one-time download, as long as you don't clean the virtual environment or change the model.

On your command line, you will get something like:

```
[{"answer":"a SaaS platform to build natural language processing applications","type":"extractive","score":0.6236580908298492,"context":"Build production-ready NLP services.  deepset Cloud is a SaaS platform to build natural language processing applications.","offsets_in_document":[{"start":55,"end":120}],"offsets_in_context":[{"start":55,"end":120}],"document_id":"4985f3369c9c9c6a6ba179a004b0af72","meta":{"name":"deepset Cloud"}}]
```

I would like to emphasize some fields of the haystack [Answer primitive](https://haystack.deepset.ai/reference/primitives#answer):

- answer: this is the piece of text that answers your submitted question. It's an extracted text (on this pipeline), which means it is identical to how the answer is presented in the document (that we sent to OpenSearchDocumentStore).

- score: the relevance, between 0 and 1, of the answer.

- offsets_in_document: where in the document the answer is located.

- document_id: the id of the document in which the answer is present.


## The end (for today)

Unfortunately, we have come to the end of this article. I would like to share much more, but it is better to keep the topics delineated 

We have built a FastAPI application that provides endpoints for some sparse haystack retrievers. However, there is much more to haystack:

- Pre-processing
- File converters
- Generator
- Summarizers
- Translators
- Rankers
- Classifiers
- and much more...

Haystack's capabilities are very extensive, covering almost all possible usages of NLP search. 


#### Useful **haystack** links

- Open an issue [here](https://github.com/deepset-ai/haystack/issues). 
- Read the [docs](https://haystack.deepset.ai/overview/intro).
- Join the [newsletter](https://haystack.deepset.ai/community/join)
- Join the [Discord server](https://discord.com/invite/VBpFzsgRVF).

I'm always around haystack on Discord and would be pleased to talk there.
