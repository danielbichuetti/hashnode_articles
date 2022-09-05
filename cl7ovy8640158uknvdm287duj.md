## NLP endpoints: Haystack plus FastAPI



In this article, we will build together a **FastAPI application** which interacts with **haystack**. It's an experiment intended to be a starting point to anyone wishing to use haystack as a service for other projects. And one of the easiest ways is through a REST API, right?
You can deploy a REST API into a physical machine, a virtual machine, a container, a serverless environment (like AWS Lambda or Azure Functions).

To the code.

## The virtual environment

Everybody loves Python! 

Who doesn't get amazed to “talk with the code”? When I'm coding in Python, I feel like talking to my computer. For someone who has been coding since when I was 12yo, you may understand about my passion. 

Python has thousands of available modules, and each day more are developed. When you need something, look for it deeper on PyPi. It's really hard not to find something. But this remembers me, one of the most famous sentences into a movie: *“With great power comes great responsibility.”* If we keep installing modules into a mixed environment, we are building the path for huge problems in dependency chain, for testing and for organization. The recommended way is to use **isolated environments**. Which can be achieved by [python-venv](https://docs.python.org/3/library/venv.html).

### Create a fresh environment.

```python
python3 -m venv venv
``` 

### Activate the virtual environment.

```python
source venv/bin/activate
``` 


## Install haystack

Now we install the outstanding **[haystack](https://haystack.deepset.ai)** NLP framework. 

[OpenSearch](https://opensearch.org/) will be our used as our Document Store. For this, we set the egg *opensearch* to instruct pip to install its related dependencies together with the framework.

```python
pip install farm-haystack[opensearch]
``` 


## Install FastAPI

[FastAPI](https://fastapi.tiangolo.com/) is a great web framework for building APIs. We install it by running:

```
pip install fastapi
```


## Create the application

Our development environment is set up. We will now begin the real application development.

### Create an empty python file

```python
touch app.py
```

### The health-check endpoint

 First, exposing a health-check endpoint is a good practice. But what is a health-check endpoint?

 It's a route that tell to external systems the current status of your application. Usual states are running correctly (green), partially (yellow) or not running correctly (red). It's crucial in some providers to let then check that your container is running correctly and don't need a restart. It's also useful by allowing you to set up alarms in monitoring tools.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}
```

This code isn't doing extended checks and returning possible warnings (yellow reasons) or errors (red reasons). But a production API really should. It's a starting point, and what it should return when not working is tightened to each application and business.

### Async capabilities

Besides using a usual definition, which is synchronous, FastAPI also supports asynchronous methods. In IO-heavy applications, asynchronous code is critical to avoid bottlenecks and improve performance. You need to add **async** into your method definition.


```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "ok"}
```

### Running your FastAPI application

Ok, you may be thinking now about how you run this application. Probably, you are considering calling “python  app.py”, right? No, FastAPI is a framework, not a server. You must use something else. Here is where [uvicorn](https://www.uvicorn.org/) is useful. There is also [gunicorn](https://gunicorn.org/), which is a WSGY server, but this is for another article. 

Install it.

```
pip install uvicorn[standard]
```


From the command line, you can now run:

```
uvicorn app:app --reload
```

The first *app* is the python filename without the extension, and the second is the FastAPI *App object*. The reload option will make uvicorn reload the file whenever it's changed. Use this only when developing, if you like real-time updates.


It will start at the default port 8000. You should see on the command-line something like:

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

The response must be:

```
{"status":"ok"}
```

Yeah, it's a JSON string. Pretty easy to build a REST API, right? This is just a small amount of FastAPI power. It's straightforward and fast.

## Deploy an OpenSearch cluster

Do you have any OpenSearch servers available?  If yes, you can jump to the next section. If not, you will love this one.

### Create an AWS OpenSearch Service

Go to [AWS console](https://signin.aws.amazon.com/signin) portal and log in. If you are not a user yet, please [sign up](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html?refid=c623d581-46f6-43a2-b227-cabbee9cd673) for a free account. 

They provide Amazon OpenSearch Service on their free tier. Yeah, 750 hours of **t3.small.search** instance monthly. It's a good starting point.

After you have logged in, find [Amazon OpenSearch Service](https://us-east-1.console.aws.amazon.com/opensearch). Then click on *Create Domain*

![Screenshot from 2022-09-04 15-06-45.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662314815773/Wk9mwnVpp.png align="left")

You will be taken to the cluster creation screen. The first field is the name you want to use for your cluster. Let's call it **haystack**.


![Screenshot from 2022-09-04 15-09-30.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662314979663/d_GLHt_LS.png align="left")

Choose the Deployment type, Development and testing, and select 1.3 (latest) as the version. 


![Screenshot from 2022-09-04 15-11-23.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315092919/RNYk4t9Lk.png align="left")

To keep setup simple and using just the free tier, let's tweak some default Data nodes options.
First, let's use only **1 AZ**, and change the **Number of nodes** to **1**. On **Instance Type**, change to **t3.small.search**. Don't change the EBS size to more than 20GB, as it's the maximum allowed for free.


![Screenshot from 2022-09-04 15-17-10.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315439686/AuhiOOKJ7.png align="left")

Scroll down until you get to the **Network** part. Usually, this is a bad idea without further tuning and control. However, for an experiment, it will be ok. We will use **Public access** for the network, which means *our cluster endpoints will be exposed to the Internet*.


![Screenshot from 2022-09-04 15-18-47.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315533432/rflTcTdTA.png align="left")

As we are exposing OpenSearch to the Internet, set up at least an **admin user**. In a real production environment, I would recommend using IAM roles, or certificates. Let's call the user “hs_admin”, and set up a password.


![Screenshot from 2022-09-04 15-23-42.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315830661/JaXxm7R6J.png align="left")

For a simpler setup, and to avoid extensive configuration, let's **turn off domain access policies** and use just fine-grained access control. **Avoid this on production**. 


![Screenshot from 2022-09-04 15-25-22.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662315939281/iS-nTTYGa.png align="left")


Now hit the Create button on the bottom of the page and wait.

You'll see a page with loading information, like this one. It will show the steps of the cluster creation process. It will take some minutes.


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

### The environments values

To connect to our OpenSearch cluster, we need to provide the cluster authentication data. Putting this directly in code is a terrible decision. Never do it. You should use secret stores, or at least environment variables.

We will now create a *.env* file in our project directory. You will fill it with the cluster address, the admin username, and the password. Remember to paste the server without the scheme(HTTPS or HTTP) and the port. These are extra parameters for haystack OpenSearchDocumentStore.


```
OPENSEARCH_SERVER=search-haystack-4luhmrfigopcmjfnvh62rtsp5q.us-east-1.es.amazonaws.com
OPENSEARCH_SERVER_PORT=443
OPENSEARCH_USER=hs_admin  
OPENSEARCH_PASSWORD=H2YsT2CkPa$$w0rd
```

Now, we will install a library that will help load environment variables during development.

```
pip install python-dotenv
```

On your app.py file, just add 

```
from dotenv import load_dotenv

load_dotenv()
OPENSEARCH_SERVER = os.getenv("OPENSEARCH_SERVER")
OPENSEARCH_SERVER_PORT = os.getenv("OPENSEARCH_SERVER_PORT")
OPENSEARCH_USER = os.getenv("OPENSEARCH_USER")
OPENSEARCH_PASSWORD = os.getenv("OPENSEARCH_PASSWORD")
```

This will load the variable from the environment, but before, dotenv will check whether they are present or not, if not, it will use the .env data.



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

There are other parameters and ways of fine-tuning this initialization, you can read more [here](https://haystack.deepset.ai/reference/document-store#opensearchdocumentstore).


### Data validation

Valid data in a REST endpoint is a need, right? We don't want anyone sending invalid data to our endpoints. We will use data validation then.

Let's define a model for the requests made to our endpoints. We will send documents, which will be JSON documents with content and a name.

We will use [pydantic](https://pydantic-docs.helpmanual.io/) for it. First, we will install it:

```
pip install pydantic
```

Then we will declare the new class:

```
class Document(BaseModel):
    name: Optional[str] = None
    content: str
```

Here there is an important point. Take care with the naming of classes on your project, to avoid overwriting other classes. You can see I used Document. But then I remembered one of the most critical [primitives in haystack](https://haystack.deepset.ai/reference/primitives): Document. Like Labels and Answers, it's a pillar of it. We will use an alias on the import like this: 

```
from haystack.schema import Document as HaystackDocument
```

Let's keep the alias, for now, just to avoid you overwriting it while still getting in touch with the best NLP framework.

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

Wow, that was easy and fast! You should see this response:

```
{"id":"e41e6aeb0ae5965a912b672664e58b7c"}
```

This is the id that haystack generated for your document. We are returning it.


### Get a document

What if we want to get this document?

Let's build an endpoint to read a document by its id:

```python
@app.get("/documents/{document_id}", status_code=200)
def get_document(document_id: str, response: Response):
    document = document_store.get_document_by_id(document_id)
    if document is None:
        response.status_code = 404
        return {"error": "Document not found"}
    return document_store.get_document_by_id(document_id)
```

We are trying to get the document from haystack OpenSearchDocumentStore, if there is not a document with this id, we will return a 404 HTTP response code, with an error message, if it exists you will get a 200 HTTP response code with the document.

```curl
curl http://localhost:8000/documents/e41e6aeb0ae5965a912b672664e58b7c
```

Our response will have a bit more information than what we first sent to create the endpoint:

```
{"content":"Haystack is an open-source framework for building search systems that work intelligently over large document collections.","content_type":"text","id":"e41e6aeb0ae5965a912b672664e58b7c","meta":"name":"My first haystack document"},"score":0.5312093733737563,"embedding":null}
```

The Document primitive of haystack documentation is [here](https://haystack.deepset.ai/reference/primitives#document). You will get a better understanding of each field there. 
By the way, haystack documentation is pretty complete. Expend some time there. It will be a pleasant learning time.


### Add more documents

```curl
curl -X POST http://localhost:8000/documents/ -H 'Content-Type: application/json' -d '{"name":"deepset Cloud", "content":"Build production-ready NLP services.  deepset Cloud is a SaaS platform to build natural language processing applications."}'
```

```curl
curl -X POST http://localhost:8000/documents/ -H 'Content-Type: application/json' -d '{"name":"FastAPI", "content":"FastAPI is a modern, fast (high-performance), web framework for building APIs with Python 3.6+ based on standard Python type hints."}'
```

### Get all documents

Now, what if we want to get all documents?

 ```python
@app.get("/documents/", status_code=200)
def get_all_document(response: Response):
    documents = document_store.get_all_documents()
    if documents is None:
        response.status_code = 404
        return {"error": "Documents not found"}
    return documents
```

We can test:

```curl
curl http://localhost:8000/documents/
```

This time our response will come with all the 3 documents we have added so far documents.

```
[{"content":"FastAPI is a modern, fast (high-performance), web framework for building APIs with Python 3.6+ based on standard Python type hints.","content_type":"text","id":"5209b538869b938ac94bf70fa0b09bd","meta":{"name":"FastAPI"},"score":null,"embedding":null},{"content":"Build production-ready NLP services.  deepset Cloud is a SaaS platform to build natural language processing applications.","content_type":"text","id":"4985f3369c9c9c6a6ba179a004b0af72","meta":{"name":"deepset Cloud"},"score":null,"embedding":null},{"content":"Haystack is an open-source framework for building search systems that work intelligently over large document collections.","content_type":"text","id":"e41e6aeb0ae5965a912b672664e58b7c","meta":{"name":"My first haystack document"},"score":null,"embedding":null}]
```

There is one important thing that you may pay attention to. **Never**, never in any production return all entities into a single call, it may be on HTTP, GRPC, SQL query. Please, never do this! It's a horrible practice. You should use *pagination techniques*. 

The example was shown above, just as a multi-document return, and because I knew there were only three documents. Imagine the damage you would cause to the servers if you had 10 million documents. 


### Querying the Document Store directly

We have extremely useful nodes and pipelines on haystack. But you can run queries directly on your Document Store, even using [custom DSL queries](https://haystack.deepset.ai/reference/document-store#baseelasticsearchdocumentstore).

Let's set up a simple “search” endpoint. 

```
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

You must keep in mind two things for this elementary search and query:

1. There are only 3 documents in our Document Store
2. SaaS keyword is present in only one document.
3. The query is extremely short and limited

If we had more documents with the keyword, the BM25 algorithm would help us with many answers, and assign scores for each answer.

So remember, haystack is much more powerful than this. We are using a tiny piece of the framework, mostly related to primitives and Document Stores. There are outstanding nodes to explore.

Furthermore, we could use a powerful filter, on its haystack implementation. It would be something similar to:

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

NOTE: this filter doesn't apply to our current application, it's just and example to understand the logic on it.


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

First, we are defining which model we will use. The company behind haystack, [deepset.ai](https://www.deepset.ai/), has made public some astonishing models on [Hugging Face](https://huggingface.co/deepset). You will find small, medium, and giant models. All of them have great quality. Please explore them. Oh, and don't forget to give them a like on Hugging Face. 

Then we have set up a BM25Retriever, which will do a sparse search for us. 

Thereafter, we are setting up the FARMReader (FARM was also developed by our friends at deepset, it's a great framework for transfer learning). 

One of the [ready-made pipelines](https://haystack.deepset.ai/components/ready-made-pipelines) is being instantiated. These pipelines fit the most common search patterns and chain haystack components. The [ExtractiveQAPipeline](https://haystack.deepset.ai/components/ready-made-pipelines#extractiveqapipeline) searches the OpenSearch document collection and extracts a piece of text (a span) that answer our question.

We call the endpoint:

```curl
curl -X POST http://localhost:8000/documents/ask -H 'Content-Type: application/json' -d '{"question":"What is deepset Cloud?"}'
```

When running this for the first time, it will take some time because haystack will download the model from the internet. This can take less or more depending on the model, deepset/roberta-base-squad2 is about 480 MB, and your connection speed. But it's a one-time download, as long as you don't clean the virtual environment. 


On your command line, you will get something like:

```
[{"answer":"a SaaS platform to build natural language processing applications","type":"extractive","score":0.6236580908298492,"context":"Build production-ready NLP services.  deepset Cloud is a SaaS platform to build natural language processing applications.","offsets_in_document":[{"start":55,"end":120}],"offsets_in_context":[{"start":55,"end":120}],"document_id":"4985f3369c9c9c6a6ba179a004b0af72","meta":{"name":"deepset Cloud"}}]
```

I would like to emphasize some fields of the haystack [Answer primitive](https://haystack.deepset.ai/reference/primitives#answer):

- answer: this is the piece of text which is the answer to your submitted question. It's an extracted text (on this pipeline), which means it's equal to how the answer is present in the document.

- score: it's the relevance of the answer.

- offsets_in_document: where in the document (as it was sent to OpenSearchDocumentStore) the answer is located.

- document_id: the document id where the answer was found.


## The end (for today)

Unfortunately, we have come to the end of this article. I would like to share more, but it's better to keep some topics separated for now.

We have built a FastAPI application that provides endpoints for some sparse haystack retrievers. There is much more on haystack:

- Pre-processing
- File converters
- Generator
- Summarizers
- Translators
- Rankers
- Classifiers
- ...

The list is yet big and covers almost all possible NLP search usage. 


#### Useful **haystack** links

- Open an issue [here](https://github.com/deepset-ai/haystack/issues). 
- Read the [docs](https://haystack.deepset.ai/overview/intro).
- Join the [newsletter](https://haystack.deepset.ai/community/join)
- Join the [Discord server](https://discord.com/invite/VBpFzsgRVF).

I'm always around haystack on Discord. I'll be pleased to talk there.















