# Haystack Technical Architecture Documentation

This document provides a comprehensive overview of the Haystack framework's technical architecture, including detailed Mermaid diagrams that illustrate the system's components, data flows, and integration patterns.

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Pipeline Architecture](#pipeline-architecture)
4. [Component Architecture](#component-architecture)
5. [Document Store Architecture](#document-store-architecture)
6. [Data Flow Patterns](#data-flow-patterns)
7. [Integration Architecture](#integration-architecture)
8. [Deployment Patterns](#deployment-patterns)

## Overview

Haystack is a production-ready, end-to-end LLM framework designed for building applications powered by Large Language Models, Transformer models, vector search, and more. The framework follows a modular, component-based architecture that enables flexible composition of NLP pipelines.

### Core Principles

- **Modular Design**: Components can be mixed and matched to create custom pipelines
- **Technology Agnostic**: Support for multiple LLM providers and models
- **Scalable**: Production-ready components for handling large-scale deployments
- **Extensible**: Easy to add custom components and integrations

## System Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph "Application Layer"
        APP[User Application]
        API[REST API / Hayhooks]
        WEB[Web Interface / deepset Studio]
    end

    subgraph "Haystack Framework"
        subgraph "Core Engine"
            PIPELINE[Pipeline Engine]
            COMPONENT[Component System]
            SERIALIZATION[Serialization Layer]
        end

        subgraph "Component Library"
            EMBEDDERS[Embedders]
            GENERATORS[Generators]
            RETRIEVERS[Retrievers]
            PROCESSORS[Preprocessors]
            ROUTERS[Routers]
            EVALUATORS[Evaluators]
        end

        subgraph "Infrastructure"
            DOCSTORE[Document Stores]
            MARSHAL[Marshalling System]
            TELEMETRY[Telemetry]
            TRACING[Distributed Tracing]
        end
    end

    subgraph "External Systems"
        subgraph "LLM Providers"
            OPENAI[OpenAI]
            COHERE[Cohere]
            HF[Hugging Face]
            AZURE[Azure OpenAI]
            LOCAL[Local Models]
        end

        subgraph "Storage Systems"
            VECTORDB[Vector Databases]
            ELASTICSEARCH[Elasticsearch]
            MEMORY[In-Memory Store]
        end

        subgraph "Data Sources"
            FILES[Files]
            APIS[APIs]
            DATABASES[Databases]
        end
    end

    APP --> PIPELINE
    API --> PIPELINE
    WEB --> PIPELINE

    PIPELINE --> COMPONENT
    COMPONENT --> EMBEDDERS
    COMPONENT --> GENERATORS
    COMPONENT --> RETRIEVERS
    COMPONENT --> PROCESSORS
    COMPONENT --> ROUTERS
    COMPONENT --> EVALUATORS

    DOCSTORE --> VECTORDB
    DOCSTORE --> ELASTICSEARCH
    DOCSTORE --> MEMORY

    GENERATORS --> OPENAI
    GENERATORS --> COHERE
    GENERATORS --> HF
    GENERATORS --> AZURE
    GENERATORS --> LOCAL

    RETRIEVERS --> DOCSTORE
    EMBEDDERS --> DOCSTORE

    PROCESSORS --> FILES
    PROCESSORS --> APIS
    PROCESSORS --> DATABASES
```

### Core Modules

```mermaid
graph LR
    subgraph "haystack.core"
        PIPELINE_MODULE[pipeline/]
        COMPONENT_MODULE[component/]
        SERIALIZATION_MODULE[serialization.py]
        TYPE_UTILS[type_utils.py]
        ERRORS[errors.py]
    end

    subgraph "haystack.components"
        EMBEDDERS_MODULE[embedders/]
        GENERATORS_MODULE[generators/]
        RETRIEVERS_MODULE[retrievers/]
        PREPROCESSORS_MODULE[preprocessors/]
        ROUTERS_MODULE[routers/]
        EVALUATORS_MODULE[evaluators/]
    end

    subgraph "haystack.document_stores"
        IN_MEMORY[in_memory/]
        TYPES[types/]
        PROTOCOLS[protocols]
    end

    subgraph "haystack.dataclasses"
        DOCUMENT[Document]
        ANSWER[Answer]
        BREAKPOINTS[Breakpoint]
    end

    PIPELINE_MODULE --> COMPONENT_MODULE
    COMPONENT_MODULE --> SERIALIZATION_MODULE
    PIPELINE_MODULE --> TYPE_UTILS
    COMPONENT_MODULE --> ERRORS

    EMBEDDERS_MODULE --> COMPONENT_MODULE
    GENERATORS_MODULE --> COMPONENT_MODULE
    RETRIEVERS_MODULE --> COMPONENT_MODULE
    PREPROCESSORS_MODULE --> COMPONENT_MODULE
    ROUTERS_MODULE --> COMPONENT_MODULE
    EVALUATORS_MODULE --> COMPONENT_MODULE

    RETRIEVERS_MODULE --> IN_MEMORY
    EMBEDDERS_MODULE --> IN_MEMORY
```

## Pipeline Architecture

### Pipeline Execution Model

```mermaid
graph TD
    START([Pipeline.run]) --> VALIDATE{Validate Pipeline}
    VALIDATE -->|Invalid| ERROR[Throw PipelineError]
    VALIDATE -->|Valid| INIT[Initialize Execution Context]

    INIT --> WARMUP[Warm Up Components]
    WARMUP --> QUEUE[Create Priority Queue]
    QUEUE --> EXECUTE[Execute Components]

    subgraph "Component Execution Loop"
        EXECUTE --> DEQUEUE[Dequeue Next Component]
        DEQUEUE --> CHECK{Component Ready?}
        CHECK -->|No| QUEUE_BACK[Queue Back with Lower Priority]
        CHECK -->|Yes| RUN_COMPONENT[Run Component]

        RUN_COMPONENT --> UPDATE[Update Component Outputs]
        UPDATE --> PROPAGATE[Propagate Data to Connected Components]
        PROPAGATE --> CHECK_COMPLETE{Pipeline Complete?}

        CHECK_COMPLETE -->|No| DEQUEUE
        CHECK_COMPLETE -->|Yes| COLLECT[Collect Final Outputs]
    end

    QUEUE_BACK --> DEQUEUE
    COLLECT --> RETURN([Return Results])

    subgraph "Error Handling"
        RUN_COMPONENT -->|Error| HANDLE_ERROR[Handle Component Error]
        HANDLE_ERROR --> BREAKPOINT{Breakpoint?}
        BREAKPOINT -->|Yes| SAVE_SNAPSHOT[Save Pipeline Snapshot]
        BREAKPOINT -->|No| PROPAGATE_ERROR[Propagate Error]
        SAVE_SNAPSHOT --> PROPAGATE_ERROR
        PROPAGATE_ERROR --> ERROR
    end
```

### Pipeline Graph Structure

```mermaid
graph LR
    subgraph "Pipeline Internal Structure"
        GRAPH[NetworkX MultiDiGraph]
        NODE_DATA[Node Data<br/>- Component Instance<br/>- Visits Count<br/>- Input/Output Data]
        EDGE_DATA[Edge Data<br/>- From Socket<br/>- To Socket<br/>- Connection Type]
    end

    subgraph "Component Priority System"
        HIGHEST[HIGHEST: 1<br/>Critical Components]
        READY[READY: 2<br/>All Inputs Available]
        DEFER[DEFER: 3<br/>Waiting for Inputs]
        DEFER_LAST[DEFER_LAST: 4<br/>Optional Inputs Missing]
        BLOCKED[BLOCKED: 5<br/>Cannot Execute]
    end

    GRAPH --> NODE_DATA
    GRAPH --> EDGE_DATA

    READY --> HIGHEST
    DEFER --> READY
    DEFER_LAST --> DEFER
    BLOCKED --> DEFER_LAST
```

### Synchronous vs Asynchronous Execution

```mermaid
sequenceDiagram
    participant User
    participant Pipeline
    participant Component1
    participant Component2
    participant Component3

    Note over User, Component3: Synchronous Pipeline Execution
    User->>Pipeline: run(inputs)
    Pipeline->>Component1: run(data)
    Component1-->>Pipeline: output1
    Pipeline->>Component2: run(output1)
    Component2-->>Pipeline: output2
    Pipeline->>Component3: run(output2)
    Component3-->>Pipeline: final_output
    Pipeline-->>User: results

    Note over User, Component3: Asynchronous Pipeline Execution
    User->>Pipeline: async_run(inputs)

    par Parallel Component Execution
        Pipeline->>Component1: async run(data)
    and
        Pipeline->>Component2: async run(data)
    end

    Component1-->>Pipeline: output1
    Component2-->>Pipeline: output2
    Pipeline->>Component3: async run(output1, output2)
    Component3-->>Pipeline: final_output
    Pipeline-->>User: results
```

## Component Architecture

### Component Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created: @component decorator
    Created --> Initialized: __init__()
    Initialized --> Validated: Pipeline validation
    Validated --> WarmingUp: warm_up()
    WarmingUp --> Ready: Ready for execution

    Ready --> Running: run()
    Running --> Completed: Return outputs
    Running --> Failed: Exception raised

    Completed --> Ready: Next execution
    Failed --> Ready: Error handled

    Ready --> [*]: Pipeline complete
    Failed --> [*]: Unhandled error
```

### Component Interface Contract

```mermaid
classDiagram
    class Component {
        <<Protocol>>
        +run(**kwargs) dict[str, Any]
        +warm_up() None
        +to_dict() dict[str, Any]
        +from_dict(dict) Component
    }

    class ComponentMetadata {
        +init_parameters: dict
        +input_types: dict
        +output_types: dict
        +component_config: ComponentConfig
    }

    class InputSocket {
        +name: str
        +type: Any
        +default_value: Any
        +is_variadic: bool
        +is_greedy: bool
    }

    class OutputSocket {
        +name: str
        +type: Any
    }

    Component ||--o{ InputSocket
    Component ||--o{ OutputSocket
    Component ||--|| ComponentMetadata

    class ConcreteComponent {
        +run(**kwargs) dict[str, Any]
        +warm_up() None
        -_component_init()
        -_validate_run_input()
    }

    ConcreteComponent --|> Component
```

### Component Types and Hierarchy

```mermaid
graph TD
    COMPONENT[Component Base] --> GENERATOR[Generator Components]
    COMPONENT --> RETRIEVER[Retriever Components]
    COMPONENT --> EMBEDDER[Embedder Components]
    COMPONENT --> PREPROCESSOR[Preprocessor Components]
    COMPONENT --> ROUTER[Router Components]
    COMPONENT --> EVALUATOR[Evaluator Components]

    GENERATOR --> OPENAI_GEN[OpenAI Generator]
    GENERATOR --> COHERE_GEN[Cohere Generator]
    GENERATOR --> HF_GEN[Hugging Face Generator]
    GENERATOR --> LOCAL_GEN[Local Generator]

    RETRIEVER --> VECTOR_RET[Vector Retriever]
    RETRIEVER --> BM25_RET[BM25 Retriever]
    RETRIEVER --> HYBRID_RET[Hybrid Retriever]

    EMBEDDER --> SENTENCE_EMB[Sentence Transformer Embedder]
    EMBEDDER --> OPENAI_EMB[OpenAI Text Embedder]
    EMBEDDER --> HF_EMB[Hugging Face Embedder]

    PREPROCESSOR --> FILE_CONV[File Converter]
    PREPROCESSOR --> TEXT_SPLIT[Text Splitter]
    PREPROCESSOR --> TEXT_CLEAN[Text Cleaner]

    ROUTER --> CONDITIONAL[Conditional Router]
    ROUTER --> META_ROUTER[Metadata Router]
    ROUTER --> LLM_ROUTER[LLM-based Router]

    EVALUATOR --> FAITHFULNESS[Faithfulness Evaluator]
    EVALUATOR --> RELEVANCE[Answer Relevance Evaluator]
    EVALUATOR --> SIMILARITY[Semantic Answer Similarity]
```

## Document Store Architecture

### Document Store Interface

```mermaid
classDiagram
    class DocumentStore {
        <<Protocol>>
        +count_documents(filters) int
        +filter_documents(filters) List~Document~
        +write_documents(documents, policy) int
        +delete_documents(document_ids) None
    }

    class SearchableDocumentStore {
        <<Protocol>>
        +_bm25_retrieval(query, filters, top_k) List~Document~
        +_embedding_retrieval(query_embedding, filters, top_k) List~Document~
    }

    class InMemoryDocumentStore {
        -storage: dict[str, Document]
        -embedding_dim: Optional[int]
        +count_documents(filters) int
        +filter_documents(filters) List~Document~
        +write_documents(documents, policy) int
        +delete_documents(document_ids) None
        +_bm25_retrieval(query, filters, top_k) List~Document~
        +_embedding_retrieval(query_embedding, filters, top_k) List~Document~
    }

    DocumentStore <|-- SearchableDocumentStore
    SearchableDocumentStore <|-- InMemoryDocumentStore

    class Document {
        +id: str
        +content: str
        +dataframe: Optional[DataFrame]
        +blob: Optional[ByteStream]
        +meta: dict[str, Any]
        +score: Optional[float]
        +embedding: Optional[List[float]]
    }

    InMemoryDocumentStore --> Document : stores
```

### Document Storage Patterns

```mermaid
graph TB
    subgraph "Document Ingestion Flow"
        RAW_DOCS[Raw Documents] --> CONVERTER[File Converter]
        CONVERTER --> PREPROCESSOR[Text Preprocessor]
        PREPROCESSOR --> SPLITTER[Document Splitter]
        SPLITTER --> EMBEDDER[Document Embedder]
        EMBEDDER --> WRITER[Document Writer]
        WRITER --> STORE[Document Store]
    end

    subgraph "Document Retrieval Flow"
        QUERY[Query] --> QUERY_EMB[Query Embedder]
        QUERY_EMB --> RETRIEVER[Document Retriever]
        RETRIEVER --> STORE
        STORE --> RANKER[Document Ranker]
        RANKER --> RESULTS[Ranked Results]
    end

    subgraph "Storage Implementations"
        STORE --> IN_MEMORY[In-Memory Store]
        STORE --> VECTOR_DB[Vector Database]
        STORE --> ELASTIC[Elasticsearch]
        STORE --> CUSTOM[Custom Store]
    end
```

## Data Flow Patterns

### RAG (Retrieval-Augmented Generation) Pipeline

```mermaid
sequenceDiagram
    participant User
    participant Pipeline
    participant QueryEmbedder
    participant Retriever
    participant DocumentStore
    participant PromptBuilder
    participant Generator
    participant LLM

    User->>Pipeline: query
    Pipeline->>QueryEmbedder: embed_query(query)
    QueryEmbedder-->>Pipeline: query_embedding

    Pipeline->>Retriever: run(query_embedding)
    Retriever->>DocumentStore: embedding_retrieval(query_embedding)
    DocumentStore-->>Retriever: relevant_documents
    Retriever-->>Pipeline: documents

    Pipeline->>PromptBuilder: run(query, documents)
    PromptBuilder-->>Pipeline: formatted_prompt

    Pipeline->>Generator: run(formatted_prompt)
    Generator->>LLM: generate(prompt)
    LLM-->>Generator: response
    Generator-->>Pipeline: answer

    Pipeline-->>User: final_answer
```

### Document Processing Pipeline

```mermaid
flowchart TD
    START([Start: Document Processing]) --> INPUT_DOCS[Input Documents]

    INPUT_DOCS --> FILE_CONVERTER[File Converter<br/>PDF, DOCX, HTML → Text]
    FILE_CONVERTER --> PREPROCESSOR[Text Preprocessor<br/>Clean, Normalize]
    PREPROCESSOR --> SPLITTER[Document Splitter<br/>Chunk into Passages]

    SPLITTER --> EMBEDDER[Document Embedder<br/>Generate Vector Embeddings]
    EMBEDDER --> WRITER[Document Writer<br/>Store in Document Store]

    WRITER --> VALIDATION{Validation<br/>Successful?}
    VALIDATION -->|Yes| SUCCESS([Documents Stored Successfully])
    VALIDATION -->|No| ERROR_HANDLER[Error Handler<br/>Log & Retry]
    ERROR_HANDLER --> WRITER

    subgraph "Parallel Processing"
        FILE_CONVERTER -.-> METADATA_EXTRACTOR[Metadata Extractor]
        METADATA_EXTRACTOR -.-> WRITER

        SPLITTER -.-> KEYWORD_EXTRACTOR[Keyword Extractor]
        KEYWORD_EXTRACTOR -.-> WRITER
    end
```

### Multi-Modal Pipeline Architecture

```mermaid
graph TB
    subgraph "Input Layer"
        TEXT_INPUT[Text Input]
        IMAGE_INPUT[Image Input]
        AUDIO_INPUT[Audio Input]
        DOCUMENT_INPUT[Document Input]
    end

    subgraph "Processing Layer"
        TEXT_PROC[Text Processor]
        IMAGE_PROC[Image Processor]
        AUDIO_PROC[Audio Processor]
        DOC_PROC[Document Processor]
    end

    subgraph "Embedding Layer"
        TEXT_EMB[Text Embedder]
        IMAGE_EMB[Image Embedder]
        AUDIO_EMB[Audio Embedder]
        MULTI_EMB[Multi-Modal Embedder]
    end

    subgraph "Storage Layer"
        UNIFIED_STORE[Unified Document Store]
    end

    subgraph "Retrieval Layer"
        SIMILARITY_SEARCH[Similarity Search]
        HYBRID_RETRIEVAL[Hybrid Retrieval]
    end

    subgraph "Generation Layer"
        CONTEXT_BUILDER[Context Builder]
        MULTI_MODAL_GEN[Multi-Modal Generator]
    end

    TEXT_INPUT --> TEXT_PROC
    IMAGE_INPUT --> IMAGE_PROC
    AUDIO_INPUT --> AUDIO_PROC
    DOCUMENT_INPUT --> DOC_PROC

    TEXT_PROC --> TEXT_EMB
    IMAGE_PROC --> IMAGE_EMB
    AUDIO_PROC --> AUDIO_EMB
    DOC_PROC --> MULTI_EMB

    TEXT_EMB --> UNIFIED_STORE
    IMAGE_EMB --> UNIFIED_STORE
    AUDIO_EMB --> UNIFIED_STORE
    MULTI_EMB --> UNIFIED_STORE

    UNIFIED_STORE --> SIMILARITY_SEARCH
    UNIFIED_STORE --> HYBRID_RETRIEVAL

    SIMILARITY_SEARCH --> CONTEXT_BUILDER
    HYBRID_RETRIEVAL --> CONTEXT_BUILDER
    CONTEXT_BUILDER --> MULTI_MODAL_GEN
```

## Integration Architecture

### External LLM Provider Integration

```mermaid
graph LR
    subgraph "Haystack Core"
        GENERATOR[Generator Component]
        CHAT_GENERATOR[Chat Generator]
        EMBEDDER[Embedder Component]
    end

    subgraph "Provider Adapters"
        OPENAI_ADAPTER[OpenAI Adapter]
        COHERE_ADAPTER[Cohere Adapter]
        HF_ADAPTER[Hugging Face Adapter]
        AZURE_ADAPTER[Azure OpenAI Adapter]
        ANTHROPIC_ADAPTER[Anthropic Adapter]
        LOCAL_ADAPTER[Local Model Adapter]
    end

    subgraph "External Services"
        OPENAI_API[OpenAI API]
        COHERE_API[Cohere API]
        HF_HUB[Hugging Face Hub]
        AZURE_OPENAI[Azure OpenAI Service]
        ANTHROPIC_API[Anthropic API]
        LOCAL_MODEL[Local Model Server]
    end

    GENERATOR --> OPENAI_ADAPTER
    GENERATOR --> COHERE_ADAPTER
    GENERATOR --> HF_ADAPTER
    GENERATOR --> AZURE_ADAPTER
    GENERATOR --> ANTHROPIC_ADAPTER
    GENERATOR --> LOCAL_ADAPTER

    CHAT_GENERATOR --> OPENAI_ADAPTER
    CHAT_GENERATOR --> COHERE_ADAPTER
    CHAT_GENERATOR --> ANTHROPIC_ADAPTER

    EMBEDDER --> OPENAI_ADAPTER
    EMBEDDER --> COHERE_ADAPTER
    EMBEDDER --> HF_ADAPTER

    OPENAI_ADAPTER --> OPENAI_API
    COHERE_ADAPTER --> COHERE_API
    HF_ADAPTER --> HF_HUB
    AZURE_ADAPTER --> AZURE_OPENAI
    ANTHROPIC_ADAPTER --> ANTHROPIC_API
    LOCAL_ADAPTER --> LOCAL_MODEL
```

### Vector Database Integration Patterns

```mermaid
graph TB
    subgraph "Haystack Integration Layer"
        DOCUMENT_STORE_PROTOCOL[DocumentStore Protocol]
        RETRIEVER_COMPONENTS[Retriever Components]
        EMBEDDER_COMPONENTS[Embedder Components]
    end

    subgraph "Vector DB Implementations"
        CHROMA[ChromaDB<br/>Document Store]
        PINECONE[Pinecone<br/>Document Store]
        WEAVIATE[Weaviate<br/>Document Store]
        QDRANT[Qdrant<br/>Document Store]
        ELASTICSEARCH[Elasticsearch<br/>Document Store]
        OPENSEARCH[OpenSearch<br/>Document Store]
    end

    subgraph "Operations"
        INDEX_OPS[Index Operations<br/>- Create Index<br/>- Update Schema<br/>- Delete Index]
        CRUD_OPS[CRUD Operations<br/>- Insert Documents<br/>- Update Documents<br/>- Delete Documents]
        SEARCH_OPS[Search Operations<br/>- Vector Search<br/>- Hybrid Search<br/>- Filtered Search]
    end

    DOCUMENT_STORE_PROTOCOL --> CHROMA
    DOCUMENT_STORE_PROTOCOL --> PINECONE
    DOCUMENT_STORE_PROTOCOL --> WEAVIATE
    DOCUMENT_STORE_PROTOCOL --> QDRANT
    DOCUMENT_STORE_PROTOCOL --> ELASTICSEARCH
    DOCUMENT_STORE_PROTOCOL --> OPENSEARCH

    RETRIEVER_COMPONENTS --> DOCUMENT_STORE_PROTOCOL
    EMBEDDER_COMPONENTS --> DOCUMENT_STORE_PROTOCOL

    CHROMA --> INDEX_OPS
    CHROMA --> CRUD_OPS
    CHROMA --> SEARCH_OPS
```

## Deployment Patterns

### Development vs Production Architecture

```mermaid
graph TB
    subgraph "Development Environment"
        DEV_APP[Development Application]
        DEV_PIPELINE[Pipeline with In-Memory Store]
        DEV_LOCAL_MODEL[Local Models]
        DEV_FILES[Local Files]
    end

    subgraph "Staging Environment"
        STAGE_APP[Staging Application]
        STAGE_PIPELINE[Pipeline with External DB]
        STAGE_CLOUD_MODEL[Cloud Models]
        STAGE_S3[S3 Storage]
    end

    subgraph "Production Environment"
        subgraph "Load Balancer Layer"
            LB[Load Balancer]
        end

        subgraph "Application Layer"
            PROD_APP1[Production App Instance 1]
            PROD_APP2[Production App Instance 2]
            PROD_APP3[Production App Instance N]
        end

        subgraph "Pipeline Layer"
            ASYNC_PIPELINE[Async Pipeline Engine]
            COMPONENT_POOL[Component Pool]
        end

        subgraph "Infrastructure Layer"
            VECTOR_DB_CLUSTER[Vector DB Cluster]
            CACHE_LAYER[Redis Cache]
            MODEL_API[Model API Gateway]
            MONITORING[Monitoring & Logging]
        end
    end

    DEV_APP --> DEV_PIPELINE
    DEV_PIPELINE --> DEV_LOCAL_MODEL
    DEV_PIPELINE --> DEV_FILES

    STAGE_APP --> STAGE_PIPELINE
    STAGE_PIPELINE --> STAGE_CLOUD_MODEL
    STAGE_PIPELINE --> STAGE_S3

    LB --> PROD_APP1
    LB --> PROD_APP2
    LB --> PROD_APP3

    PROD_APP1 --> ASYNC_PIPELINE
    PROD_APP2 --> ASYNC_PIPELINE
    PROD_APP3 --> ASYNC_PIPELINE

    ASYNC_PIPELINE --> COMPONENT_POOL
    COMPONENT_POOL --> VECTOR_DB_CLUSTER
    COMPONENT_POOL --> CACHE_LAYER
    COMPONENT_POOL --> MODEL_API

    VECTOR_DB_CLUSTER --> MONITORING
    CACHE_LAYER --> MONITORING
    MODEL_API --> MONITORING
```

### Microservices Deployment with Hayhooks

```mermaid
graph TB
    subgraph "Client Applications"
        WEB_APP[Web Application]
        MOBILE_APP[Mobile Application]
        API_CLIENT[API Client]
    end

    subgraph "API Gateway Layer"
        API_GATEWAY[API Gateway]
        RATE_LIMITER[Rate Limiter]
        AUTH[Authentication]
    end

    subgraph "Hayhooks Services"
        HAYHOOK1[RAG Pipeline Service]
        HAYHOOK2[Document Processing Service]
        HAYHOOK3[Evaluation Service]
        HAYHOOK4[Chat Service]
    end

    subgraph "Haystack Pipelines"
        RAG_PIPELINE[RAG Pipeline]
        DOC_PIPELINE[Document Processing Pipeline]
        EVAL_PIPELINE[Evaluation Pipeline]
        CHAT_PIPELINE[Chat Pipeline]
    end

    subgraph "Shared Infrastructure"
        SHARED_VECTOR_DB[Shared Vector Database]
        SHARED_CACHE[Shared Cache Layer]
        SHARED_MONITORING[Monitoring & Observability]
        MESSAGE_QUEUE[Message Queue]
    end

    WEB_APP --> API_GATEWAY
    MOBILE_APP --> API_GATEWAY
    API_CLIENT --> API_GATEWAY

    API_GATEWAY --> RATE_LIMITER
    RATE_LIMITER --> AUTH

    AUTH --> HAYHOOK1
    AUTH --> HAYHOOK2
    AUTH --> HAYHOOK3
    AUTH --> HAYHOOK4

    HAYHOOK1 --> RAG_PIPELINE
    HAYHOOK2 --> DOC_PIPELINE
    HAYHOOK3 --> EVAL_PIPELINE
    HAYHOOK4 --> CHAT_PIPELINE

    RAG_PIPELINE --> SHARED_VECTOR_DB
    DOC_PIPELINE --> SHARED_VECTOR_DB
    EVAL_PIPELINE --> SHARED_CACHE
    CHAT_PIPELINE --> SHARED_CACHE

    HAYHOOK1 --> MESSAGE_QUEUE
    HAYHOOK2 --> MESSAGE_QUEUE

    SHARED_VECTOR_DB --> SHARED_MONITORING
    SHARED_CACHE --> SHARED_MONITORING
    MESSAGE_QUEUE --> SHARED_MONITORING
```

---

This architecture documentation provides a comprehensive technical overview of the Haystack framework. The diagrams illustrate the modular design, component interactions, data flows, and deployment patterns that make Haystack a powerful and flexible platform for building LLM applications.

For more information, refer to the [official Haystack documentation](https://docs.haystack.deepset.ai/) and the [API reference](https://docs.haystack.deepset.ai/reference/).
