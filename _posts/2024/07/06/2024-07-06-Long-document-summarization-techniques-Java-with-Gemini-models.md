# Long Document Summarization Techniques with Java and Gemini models

Suppose your organization has a large number of documents, in various formats, and you, a Java developer, are tasked to efficiently summarize the content of each document.

While summarizing any document with just a few paragraphs is a simple task, there are several challenges to overcome when summarizing large documents with many pages.

Generative AI is top of mind for both developer and business stakeholder and you want to explore how Large Language Models (LLMs) can help you with large document summarization, a complex use-case with universal applicability in the enterprise.

As a Java developer, you're adept at building robust, scalable, high-performance applications. While Python dominates the ML and NLP landscape, Java is the backbone of enterprise software for a long time. It's strength in enterprise systems makes it an ideal platform for integrating advanced NLP techniques. With LLM usage, you can now add powerful, AI-driven insights, to your Java applications, bridging the gap between traditional enterprise software and cutting-edge machine learning capabilities.

This blog post explores various summarization techniques using LLMs, leaving you with practical information and a codebase with ready-to-test Java examples. The objective is to enable you with both theoretical knowledge and hands-on skills for effective document summarization.

We'll be leveraging [Vertex AI](https://cloud.google.com/vertex-ai?e=48754805&hl=en) with the latest [Gemini models](https://deepmind.google/technologies/gemini/) and the open-source [Langchin4J](https://docs.langchain4j.dev/) LLM orchestration framework. 
## Why consider LLMs for text summarization

LLMs offer a number of advantages over traditional extractive summarization methods:
- Context understanding: can grasp complex nuances in text, producing more coherent and relevant summaries
- Abstractive capabilities: will generate new sentences capturing the essence of the original text
- Flexibility: can be fine-tuned for specific domains or styles of summarization
- Multilingual support: many LLMs work across multiple languages, with versatility important for global applications
## Text Summarization Techniques

We'll explore in detail the following three summarization techniques in this blog post
* **Prompt Stuffing** - pass in the content of the entire document as a prompt in the LLM's content window
* **Map-reduce** - split the document into smaller (potentially overlapping) segments, summarize each segment in parallel, then summarize the individual summaries in asecond and final step
* **Refine iteratively** - split the document as in map-reduce, summarize the first segment, then ask the LLM to refine the initial summary iteratively with the text from the following segment, to the end of the text.
## Before you start
The sample code uses Java 21 throughout. If not already installed, use the [following instructions](https://github.com/GoogleCloudPlatform/serverless-production-readiness-java-gcp/tree/main/ai-patterns/summarization-langchain4j#setup-java-ecosystem) to set it up.

Sample [documentation](https://github.com/GoogleCloudPlatform/serverless-production-readiness-java-gcp/blob/main/ai-patterns/summarization-langchain4j/README.md) provides details for [cloning the repository](https://github.com/GoogleCloudPlatform/serverless-production-readiness-java-gcp/blob/main/ai-patterns/summarization-langchain4j/README.md#clone-the-code), setting the [environment up](https://github.com/GoogleCloudPlatform/serverless-production-readiness-java-gcp/blob/main/ai-patterns/summarization-langchain4j/README.md#summarization-techniques---langchain4j-vertexai-gemini) and [authenticating to Vertex AI](https://cloud.google.com/vertex-ai/docs/authentication)

## Loading and splitting the document
Before summarization can be started, you need to load the document, then, depending of your summarization choice, split up the content into smaller segments that can fit into the context window for your chosen LLM.

The the latest multimodal Gemini models in Vertex AI have [very large context windows](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models), up to 2M tokens, however you will have to adapt to the context window for LLM of your choice.

Langchain4J offers a number of out-of-the-box [Document Loaders](https://docs.langchain4j.dev/tutorials/rag#document-loader), [Document Parsers](https://docs.langchain4j.dev/tutorials/rag#document-parser) and  [DocumentSplitters](https://docs.langchain4j.dev/tutorials/rag#document-splitter) . It is very important to explore which one would yield the best results for your use-case.

The codebase for this blog loads the documents from the test folder using a `FileSystemDocumentLoader` and the `TextDocumentParser`, with the sample documents in text format. 

For text splitting, the `DocumentByParagraphSplitter` is being used. It splits the provided Document into paragraphs and attempts to fit as many paragraphs as possible into a single TextSegment, adhering to the limit set for the segment size. The splitter allows you to specify an **overlap window for segments**, with the benefits discussed later in the post.

Choosing **the right segment size** is an exercise dependent on the length of the context window for the LLM of choice.

```java
// load and parse the document  
Document document = loadDocument(resource, new TextDocumentParser());  
  
// Overlap window size between chunks set to OVERLAP_SIZE - can be configured  
// from 0 - text.length()  
DocumentSplitter splitter = new DocumentByParagraphSplitter(CHUNK_SIZE, OVERLAP_SIZE);  
List<TextSegment> chunks = splitter.split(document);
```

## LLM conversation inputs

@SystemMessage and @UserMessage are commonly used in the context of prompting and interacting with Large Language Models (LLMs)

 [@SystemMessage](https://docs.langchain4j.dev/tutorials/ai-services#systemmessage) is used to set the context, or role of the AI models, and is usually not visible to the user. We will use for system instructions the same @SystemMessage whenever the AI Service is invoked.

[@UserMessage](https://docs.langchain4j.dev/tutorials/ai-services#usermessage) represents the actual input from the human user interacting with the AI. It's the question, prompt, or statement that the user wants the AI to respond to.

@SystemMesage and @UserMessage can be provided directly as Strings or loaded from a prompt template from resources: `SystemMessage(fromResource = "my-system-prompt-template.txt")` or `@UserMessage(fromResource = "my-user-template.txt")`
## #1: Prompt Stuffing

Stuffing is the simplest summarization technique, as you need only to pass in the content of the entire document  as a prompt in the LLM's content window. However, as prompts for LLMs are token-count-limited, different techniques need to be used for large documents, depending on the size of the content window. 

Google's Gemini models have very large context windows, making them an easy choice summarizing large documents. (see limits [here](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/models))

```java
public interface StuffingSummarizationAssistant {  
    @SystemMessage("""  
    You are a helpful AI assistant.    
    You are an AI assistant that helps people summarize information.    
    Your name is Gemini    
    You should reply to the users request with your name and also in the style 
    of a literary critic    
    Strictly ignore Project Gutenberg & ignore copyright notice in summary 
    output.    
    """)  
    @UserMessage("""  
    Please provide a concise summary in strictly no more 
    than 10 one sentence bullet points,    
    starting with an introduction and ending with a conclusion, 
    of the following text
                  TEXT: {{your-content}}    
    """)  
    String summarize(@V("content") String content);  
}

...
// summarize the document with the help of the StuffingSummarizationAssistant 
StuffingSummarizationAssistant assistant = AiServices.create(StuffingSummarizationAssistant.class, chatModel);  
String response = assistant.summarize(document.text());
...				 
```
#### Pros:
* Single call to the LLM required to summarize the text, most likely faster than with multiple summarization calls
* Model has access to the entire document content at once, potentially resulting in better summary results
#### Cons
* Stuffing is applicable only as long as the entire document content can fit into the LLM context window
## #2: Map-Reduce

Map-reduce is more intricate than prompt stuffing and implements a multi-stage summarization, as you split the document into smaller (optionally overlapping) segments, summarize each segment in parallel, then summarize the individual summaries in a second and final step.

In this method, you need to prepare two user prompt templates, one for the initial segment summarization step and another for the final combine step. The system instructions remain the same across all LLM calls.
#### Splitting the text and summarizing individual segments (the "map" step)
You'll be using the following @UserMessage:
```java
public interface ChunkSummarizationAssistant {
	@SystemMessage(fromResource = "my-system-prompt-template.txt")
	@UserMessage("""  
	Taking the following context delimited by triple backquotes into consideration   
	'''{{context}}'''    
	Write a concise summary of the following text delimited by triple backquotes.  
	'''{{your-content}}'''  
	Output starts with CONCISE SUB-SUMMARY:  
	""")
	String summarize(@V("context") String context, @V("content") String content);
}

...
ChunkSummarizationAssistant assistant = AiServices.create(ChunkSummarizationAssistant.class, chatModel);  
String response = assistant.summarize(context.toString(), segment);
```

Map-reduce allows you to parallelize the individual segment summarization steps, as they are independent of each other:
```java
List<CompletableFuture<Map<Integer, String>>> futures = new ArrayList<>();  
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();  
Map<Integer, String> resultMap = new TreeMap<>(); // TreeMap to automatically sort by key  
  
for(int i = 0; i < segments.size(); i++) {  
    int index = i;  
    CompletableFuture<Map<Integer, String>> future = CompletableFuture  
        .supplyAsync(() -> summarizeChunk(index, segments.get(index).text()), executor);  
    futures.add(future);  
}  
  
// Wait for all futures to complete and collect the results in resultMap  
CompletableFuture<Void> allDone = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))  
        .thenAccept(v -> futures.forEach(f -> f.thenAccept(resultMap::putAll)));  
  
allDone.get(); // Wait for all processing to complete
```

A **key factor for improving summarization results** is the concept of **overlapping segments**. Splitting a document by a specific segment size, even if done with utility classes which will split the text cleanly in paragraphs, then fit entire sentences into the remaining segment space, is arbitrary from a summarization perspective. 

Providing the ability for segments to overlap up to a specified overlap size can yield better summarization results by preserving more contexts between the individual segments. 

Please consider experimenting with different segment and overlap sizes for your respective summarization use-cases.

Please **note** that the degree to which you can parallelize LLM calls depends on whether the rate limit of API calls per minute imposed by the LLM!
#### Summary of summaries (the "reduce" part)
With all individual summaries on hand, you can move on to the second and final step, the summarization of the individual summaries.

You would be using a different @UserMessage in this step:
```java
public interface FinalSummarizationAssistant {  
    @SystemMessage(fromResource = "my-system-prompt-template.txt")  
    @UserMessage("""  
    Please provide a concise summary in strictly no more than 10 one sentence bullet points,    
    starting with an introduction and ending with a conclusion, 
    of the following text delimited by triple backquotes.
          '''Text:{{your-content}}'''  
      Output starts with SUMMARY:  
    """)  
    String summarize(@V("content") String content);  
}
...
FinalSummarizationAssistant assistant = AiServices.create(FinalSummarizationAssistant.class, chatModel);  
String response = assistant.summarize(content);
```
#### Pros:
* Large documents can be summarized even with LLMs with smaller context windows
* Parallel processing leads to reduced summarization latency
* Overlapping segments can improve summarizaiton accuracy
#### Cons
* Multiple LLM calls are required
* There can be context loss due to arbitrary text splitting
* Overlapping segments can slightly increase latency and create larger input text
## #3: Refine

The refine method is an alternative to map-reduce to handle large document summarization. You split the document as in map-reduce, summarize the first segment, then ask the LLM to refine the initial summary iteratively with the added text from the following segment, to the end of the text.

This approach ensures a that the summary is both comprehensive, as well as accurate, as it takes into consideration the context of the previous segment(s).

You would be using the same @UserMessages illustrated in the two steps in `map-reduce`: `ChunkSummarizationAssistant` and `FinalSummarizationAssistant`.
```java
// process each individual chunk in order  
// summary refined in each step by adding the summary of the current chunk  
long start = System.currentTimeMillis();  
StringBuilder context = new StringBuilder();  
chunks.forEach(segment -> summarizeChunk(context, segment.text()));  
  
// process the final summary  of the text  
String output = buildFinalSummary(context.toString());
```
#### Pros:
* Large documents can be summarized even with LLMs with smaller context windows
* Context is preserved between segments, improving summarization accuracy and completeness
* Overlapping segments can improve summarization accuracy even further
#### Cons
* Multiple LLM calls are required
* Must be executed iteratively and does not lend itself to parallel processing, due to the interdependent nature of the individual segments and their associated context
* Latency significantly higher than map-reduce
## Summary

In this blog post, we have explored different programmatic summarization techniques for large documents using Google's Gemini LLM, as an advanced use-case for generative AI in enterprise software. 

You have a full codebase available, with practical examples, demonstrating how to implement these techniques efficiently in Java.

As an enterprise Java developer, you now have powerful options to leverage LLMs and add AI-driven insights to your applications, potentially transforming how you handle document analysis and summarization.

The field of AI-powered document summarization is rapidly evolving, with new models and techniques emerging regularly. Stay tuned for future developments that could further enhance these capabilities.

Don't hesitate to reach out at [@ddobrin](https://twitter.com/ddobrin) for feedback, questions or to discuss new summarization techniques.
