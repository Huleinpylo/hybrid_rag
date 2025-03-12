# Detailed Self-RAG Workflow Explanation

## Overview
The Self-RAG (Retrieval Augmented Generation with Self-Verification) system is an advanced question-answering framework built on LangGraph that employs multiple verification steps to ensure high-quality, factual responses. It addresses common LLM challenges like hallucinations and irrelevant answers through a sophisticated workflow of retrieval, evaluation, and generation.

## 1. Question Processing

### Language Detection
- **Implementation**: Uses the `langdetect` library to analyze input text patterns
- **Languages Supported**: Currently optimized for English and French, with English as fallback
- **Impact**: Enables language-specific prompts, document evaluation, and response generation
- **Code Implementation**: The `detect_language()` method analyzes the text and returns language code ('en' or 'fr')

### Initial Query Understanding
- **Query Analysis**: Preserves the original user intent while preparing for document retrieval
- **Entry Point**: The `ask_with_selfrag()` method initiates the workflow with the question
- **Parameter Handling**: Processes model selection, temperature settings, and top-k document settings
- **State Management**: Creates an initial state with the question for the LangGraph workflow

## 2. Document Retrieval and Evaluation

### Document Retrieval
- **Hybrid Search Implementation**: Combines vector similarity (semantic) and BM25 (keyword) search via the orchestrator
- **Search Process**:
  - The `retrieve()` node in the graph calls the orchestrator's search method
  - Document chunks are retrieved with their metadata (document_id, chunk_id)
  - Documents are cached for repeat queries to improve performance
- **Fallback Mechanisms**:
  - If orchestrator search fails, attempts direct access to HybridSearchEngine
  - If all retrieval methods fail, creates synthetic context with general information
  - Special handling for synthetic contexts during later verification steps

### Document Relevance Grading
- **Evaluation Process**: Each document undergoes LLM-based relevance assessment in the `grade_documents()` node
- **Grading Criteria**: Documents are evaluated on their semantic relevance to the question
- **Prompt Engineering**:
  - Language-specific prompts guide the LLM to evaluate document relevance
  - System prompt instructs: "It does not need to be a stringent test. The goal is to filter out erroneous retrievals."
  - Binary classification: 'yes' indicates relevance, 'no' indicates irrelevance
- **Implementation**: `grade_document_relevance()` method handles document evaluation with Ollama
- **Failsafe**: If grading fails, documents are assumed relevant (cautious inclusion)

### Decision Point
- **Conditional Execution**: The `decide_to_generate()` method determines next steps based on document filtering
- **Branch Logic**:
  - If at least one document passes relevance grading → proceed to answer generation
  - If all documents are filtered out → invoke query transformation
- **Execution Flow**: LangGraph conditional edges manage the workflow direction

## 3. Answer Generation and Verification

### Answer Generation
- **Context Preparation**: Relevant documents are concatenated with proper formatting
- **Generation Process**: 
  - The `generate()` node calls `generate_answer()` to create a response based on context
  - Uses language-specific system prompts to guide generation
  - Formats context with document numbering for clear reference
- **Implementation Details**:
  - Language-specific user message templating
  - Ollama model with specified temperature for controlled creativity
  - Document preparation with proper structure for context window

### Hallucination Check
- **Verification Implementation**: `grade_hallucination()` method evaluates if the generated answer is grounded in documents
- **Special Handling**:
  - Synthetic contexts receive automatic "yes" scores to bypass strict verification
  - Regular contexts undergo detailed assessment of factual grounding
- **Prompt Engineering**:
  - System prompt: "You are a grader assessing whether an LLM generation is grounded in / supported by a set of retrieved facts."
  - Binary classification: 'yes' indicates grounded, 'no' indicates hallucination
- **Conditional Flow**: If hallucinations detected, regeneration is attempted instead of proceeding

### Question Alignment Check
- **Purpose**: Ensures answer addresses the original question, not just factually correct
- **Implementation**: The `grade_answer()` method evaluates question-answer alignment
- **Evaluation Criteria**:
  - Whether the answer adequately addresses the question intent
  - Completeness of the response relative to the question
- **Prompt Engineering**:
  - System prompt: "You are a grader assessing whether an answer addresses / resolves a question."
  - Binary classification: 'yes' indicates well-addressed, 'no' indicates misalignment
- **Decision Point**: Determines whether to accept the answer or refine the query

## 4. Query Refinement

### Query Transformation
- **Implementation**: The `transform_query()` node calls `rewrite_question()` method
- **Rewriting Strategies**:
  - Semantic expansion of the original query
  - Addition of relevant keywords to improve retrieval
  - Clarification of ambiguous terms
- **Technical Approach**:
  - LLM-based query rewriting with explicit prompting
  - Fallback mechanism using spaCy and WordNet for language-specific synonym expansion
  - For English: WordNet synsets provide synonyms
  - For French: Manual mapping of common terms to synonyms
- **Integration**: Rewritten query restarts the retrieval process creating a feedback loop

### Feedback Loop Implementation
- **Workflow Control**: LangGraph edges connect nodes to form a controlled execution flow
- **Loop Conditions**:
  - Loop #1: No relevant documents → transform query → retrieve again
  - Loop #2: Answer doesn't address question → transform query → retrieve again
  - Loop #3: Answer contains hallucinations → regenerate with same documents
- **Termination Condition**: Generation is grounded in facts AND addresses the question
- **Safeguards**: Internal retry limits prevent infinite loops

## 5. Technical Implementation Details

### Architecture Components
- **StateGraph**: LangGraph's StateGraph manages the workflow state transitions
- **GraphState**: TypedDict tracking question, documents, generation, and settings
- **Prompts Management**: Language-specific prompts loaded from JSON files with fallbacks
- **Error Handling**: Comprehensive try-except blocks with detailed logging
- **Model Connection**: LangChain's ChatOllama interfaces with Ollama server

### LLM Integration
- **Model Usage**: Leverages local Ollama models (default is configurable)
- **Connection Management**: Tests connectivity at initialization and handles retries
- **Query Parameters**:
  - Temperature: Controls randomness/creativity in generations
  - Model selection: Configurable based on task requirements
  - Retry mechanism: Implements exponential backoff for connection issues

### Error Recovery and Fallbacks
- **Connection Failures**: Automatic retries with exponential backoff
- **Grading Errors**: Conservative defaults (include documents, assume grounded)
- **Generation Errors**: Fallback to simple document excerpts
- **Document Retrieval Failures**: Synthetic context generation
- **Query Rewriting Failures**: Keyword-based expansion as backup

### Performance Considerations
- **Caching**: Implements query result caching to avoid redundant processing
- **Logging**: Comprehensive logging for performance monitoring and debugging
- **Host Configuration**: Allows dynamic switching of Ollama hosts

## 6. Extensibility and Customization

### Language Support
- **Current Languages**: English and French fully supported
- **Extensibility**: Framework for adding more languages by extending prompt templates
- **Localization**: Language-specific prompts, evaluation criteria, and fallback mechanisms

### Model Flexibility
- **Model Selection**: Compatible with various Ollama models
- **Configuration**: Temperature and other generation parameters configurable
- **Selection API**: Additional functions to list available models and pull new ones

### Custom Prompts
- **Prompt Management**: JSON-based prompt storage with language-specific versions
- **Default Creation**: Automatic generation of default prompts if none exist
- **Customization Points**: Each stage (retrieval, generation, verification) has specific prompts