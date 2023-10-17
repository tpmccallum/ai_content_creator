# README

This is a set of instructions to:
- harvest many websites,
- fetch pages from each website,
- use LangChain question answering (powered by the Open Source Llama 2 Model from Facebook AI) on those individual web pages, and then
- answer questions 

# Prerequisites

One or more websites that adhere to the [sitemap protocol](https://www.sitemaps.org/protocol.html) in this code we access [fermyon.com](https://www.fermyon.com/sitemap.xml), [developer.fermyon.com](https://developer.fermyon.com/sitemap.xml) and [component-model.bytecodealliance.org](https://component-model.bytecodealliance.org/sitemap.xml).

A [HuggingFace account](https://huggingface.co/) (that will provide an access token)

# Bash

```bash
!pip install langchain
!pip install huggingface_hub
!pip install sentence_transformers
!pip install faiss-cpu
!pip install unstructured
!pip install chromadb
!pip install Cython
!pip install tiktoken
!pip install unstructured[local-inference]
```

# Python

```python
import os
import requests
os.environ["HUGGINGFACEHUB_API_TOKEN"] = "your_hugging_face_token"

from langchain.text_splitter import CharacterTextSplitter
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.vectorstores import FAISS
from langchain.chains.question_answering import load_qa_chain
from langchain import HuggingFaceHub
from langchain.indexes import VectorstoreIndexCreator
from langchain.chains import RetrievalQA
from langchain.document_loaders import UnstructuredURLLoader

# Use standard library to fetch contents of URLs
import requests
# Use standard library to parse fetched sitemaps
import xml.dom.minidom as minidom

# Fetch sitemap's text of each site that is to be indexed
fermyon_website_sitemap = requests.get('https://www.fermyon.com/sitemap.xml', allow_redirects=True).text
fermyon_documentation_sitemap = requests.get('https://developer.fermyon.com/sitemap.xml', allow_redirects=True).text
component_model_documentation_sitemap = requests.get('https://component-model.bytecodealliance.org/sitemap.xml', allow_redirects=True).text

# Parse each sitemap's text to obtain list of pages
parsed_fermyon_website_sitemap_document = minidom.parseString(fermyon_website_sitemap)
parsed_fermyon_documentation_sitemap_document = minidom.parseString(fermyon_documentation_sitemap)
parsed_component_model_documentation_sitemap_document = minidom.parseString(component_model_documentation_sitemap)

# Cherry pick just the loc elements from the XML
fermyon_website_sitemap_loc_elements = parsed_fermyon_website_sitemap_document.getElementsByTagName('loc')
fermyon_documentation_sitemap_loc_elements = parsed_fermyon_documentation_sitemap_document.getElementsByTagName('loc')
component_model_documentation_sitemap_loc_elements = parsed_component_model_documentation_sitemap_document.getElementsByTagName('loc')

# Declare blank lists of pages for each site
fermyon_website_page_urls = []
fermyon_documentation_page_urls = []
component_model_documentation_page_urls = []

# Iterate over loc elements (of each sitemap) and add to that site's list of pages
for fermyon_website_sitemap_loc_element in fermyon_website_sitemap_loc_elements:
    fermyon_website_page_urls.append(fermyon_website_sitemap_loc_element.toxml().removesuffix("</loc>").removeprefix("<loc>"))
for fermyon_documentation_sitemap_loc_element in fermyon_documentation_sitemap_loc_elements:
    fermyon_documentation_page_urls.append(fermyon_documentation_sitemap_loc_element.toxml().removesuffix("</loc>").removeprefix("<loc>"))
for component_model_documentation_sitemap_loc_element in component_model_documentation_sitemap_loc_elements:
    component_model_documentation_page_urls.append(component_model_documentation_sitemap_loc_element.toxml().removesuffix("</loc>").removeprefix("<loc>"))

URLs = fermyon_website_page_urls + fermyon_documentation_page_urls + component_model_documentation_page_urls

print("Number of page to process is {}\n First page to process is {} and the last page to process is {}".format(len(URLs), URLs[0], URLs[len(URLs) - 1]))

text_to_remove = "rss"
filtered_list = [item for item in URLs if text_to_remove not in item]

from langchain.document_loaders import UnstructuredURLLoader

loader2 = [UnstructuredURLLoader(urls=filtered_list)]

index2 = VectorstoreIndexCreator(
    embedding=HuggingFaceEmbeddings(),
    text_splitter=CharacterTextSplitter(chunk_size=1000, chunk_overlap=0)).from_loaders(loader2)

llm2=HuggingFaceHub(repo_id="declare-lab/flan-alpaca-large", model_kwargs={"temperature":0, "max_length":512})

from langchain.chains import RetrievalQA
chain = RetrievalQA.from_chain_type(llm=llm2,
                                    chain_type="stuff",
                                    retriever=index2.vectorstore.as_retriever(),
                                    input_key="question")
chain.run('What is the WebAssembly Component Model?')
```

Returns

"The WebAssembly Component Model is a broad-reaching architecture for building interoperable Wasm libraries, applications, and environments."
