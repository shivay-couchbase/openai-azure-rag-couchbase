Here's a README.md for the final.py file:

# Chat with PDF using Langchain, Couchbase & Azure OpenAI

A Streamlit application that allows users to chat with PDF documents using RAG (Retrieval Augmented Generation) powered by Couchbase Vector Store and Azure OpenAI.

## Features

- Upload and process PDF documents
- Chat interface with dual responses:
  - RAG-based answers using document context
  - Pure LLM answers without context
- Secure authentication system
- Vector storage using Couchbase
- Azure OpenAI integration for embeddings and chat
- Streaming responses for better user experience

## Prerequisites

- Python 3.7+
- Couchbase database
- Azure OpenAI API access
- Required Python packages:
```text
langchain-couchbase
langchain-openai
streamlit
PyPDF2
```

## Environment Variables

Create a `secrets.toml` file with the following variables:

```toml
AUTH_ENABLED = "True/False"
LOGIN_PASSWORD = "your_password"
DB_CONN_STR = "your_couchbase_connection_string"
DB_USERNAME = "your_username"
DB_PASSWORD = "your_password"
DB_BUCKET = "your_bucket"
DB_SCOPE = "your_scope"
DB_COLLECTION = "your_collection"
INDEX_NAME = "your_index_name"
AZURE_OPENAI_EMBEDDING_DEPLOYMENT = "your_embedding_deployment"
AZURE_OPENAI_API_KEY = "your_azure_openai_key"
AZURE_OPENAI_ENDPOINT = "your_azure_endpoint"
AZURE_OPENAI_CHAT_DEPLOYMENT = "your_chat_deployment"
```

## How It Works

1. **Authentication**: Optional password protection for the application

```121:141:final.py

    AUTH_ENABLED = parse_bool(os.getenv("AUTH_ENABLED", "False"))

    if not AUTH_ENABLED:
        st.session_state.auth = True
    else:
        # Authorization
        if "auth" not in st.session_state:
            st.session_state.auth = False

        AUTH = os.getenv("LOGIN_PASSWORD")
        check_environment_variable("LOGIN_PASSWORD")

        # Authentication
        user_pwd = st.text_input("Enter password", type="password")
        pwd_submit = st.button("Submit")

        if pwd_submit and user_pwd == AUTH:
            st.session_state.auth = True
        elif pwd_submit and user_pwd != AUTH:
            st.error("Incorrect password")
```


2. **PDF Processing**: Uploads and chunks PDFs for vector storage

```33:51:final.py
def save_to_vector_store(uploaded_file, vector_store):
    """Chunk the PDF & store it in Couchbase Vector Store"""
    if uploaded_file is not None:
        temp_dir = tempfile.TemporaryDirectory()
        temp_file_path = os.path.join(temp_dir.name, uploaded_file.name)

        with open(temp_file_path, "wb") as f:
            f.write(uploaded_file.getvalue())
            loader = PyPDFLoader(temp_file_path)
            docs = loader.load()

        text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1500, chunk_overlap=150
        )

        doc_pages = text_splitter.split_documents(docs)

        vector_store.add_documents(doc_pages)
        st.info(f"PDF loaded into vector store in {len(doc_pages)} documents")
```


3. **Vector Store**: Uses Couchbase for storing document embeddings

```54:72:final.py
@st.cache_resource(show_spinner="Connecting to Vector Store")
def get_vector_store(
    _cluster,
    db_bucket,
    db_scope,
    db_collection,
    _embedding,
    index_name,
):
    """Return the Couchbase vector store"""
    vector_store = CouchbaseVectorStore(
        cluster=_cluster,
        bucket_name=db_bucket,
        scope_name=db_scope,
        collection_name=db_collection,
        embedding=_embedding,
        index_name=index_name,
    )
    return vector_store
```


4. **RAG Implementation**: Combines vector search with Azure OpenAI

```215:220:final.py
        chain = (
            {"context": retriever, "question": RunnablePassthrough()}
            | prompt
            | llm
            | StrOutputParser()
        )
```


5. **Chat Interface**: Streamlit-based UI with streaming responses

```285:341:final.py
        if "messages" not in st.session_state:
            st.session_state.messages = []
            st.session_state.messages.append(
                {
                    "role": "assistant",
                    "content": "Hi, I'm a chatbot who can chat with the PDF. How can I help you?",
                    "avatar": "🤖",
                }
            )

        # Display chat messages from history on app rerun
        for message in st.session_state.messages:
            with st.chat_message(message["role"], avatar=message["avatar"]):
                st.markdown(message["content"])

        # React to user input
        if question := st.chat_input("Ask a question based on the PDF"):
            # Display user message in chat message container
            st.chat_message("user").markdown(question)

            # Add user message to chat history
            st.session_state.messages.append(
                {"role": "user", "content": question, "avatar": "👤"}
            )

            # Add placeholder for streaming the response
            with st.chat_message("assistant", avatar=couchbase_logo):
                # Get the response from the RAG & stream it
                # In order to cache the response, we need to invoke the chain and cache the response locally as OpenAI does not support it yet
                # Ref: https://github.com/langchain-ai/langchain/issues/9762

                rag_response = chain.invoke(question)

                st.write_stream(stream_string(rag_response))

            st.session_state.messages.append(
                {
                    "role": "assistant",
                    "content": rag_response,
                    "avatar": couchbase_logo,
                }
            )

            # Get the response from the pure LLM & stream it
            pure_llm_response = chain_without_rag.invoke(question)

            # Add placeholder for streaming the response
            with st.chat_message("ai", avatar="🤖"):
                st.write_stream(stream_string(pure_llm_response))

            st.session_state.messages.append(
                {
                    "role": "assistant",
                    "content": pure_llm_response,
                    "avatar": "🤖",
                }
            )
```


## Usage

1. Start the application:
```bash
streamlit run final.py
```

2. If authentication is enabled, enter the password

3. Upload a PDF document using the sidebar

4. Start chatting with your document

## Response Types

- 🤖 Responses are generated using pure LLM (Azure OpenAI)
- [Couchbase Logo] Responses are generated using RAG with document context

## Caching

The application implements caching for:
- Vector store connections
- Cache connections
- Couchbase connections

This improves performance and reduces API calls.

## Error Handling

The application includes comprehensive error handling for:
- Environment variable validation
- Database connections
- PDF processing
- Vector store operations

## Contributing

Feel free to submit issues and enhancement requests.

## License

[Add your license information here]
