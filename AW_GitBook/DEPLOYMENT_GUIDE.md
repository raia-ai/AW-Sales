# GitBook Framework Deployment Guide

## Overview

This guide provides step-by-step instructions for deploying the A+W Software GitBook framework to various platforms and integrating it with AI vector stores.

## Deployment Options

### Option 1: GitBook Platform (Recommended)

#### Prerequisites
- GitBook account (free or paid)
- Git repository (GitHub, GitLab, or Bitbucket)

#### Steps
1. **Create a Git Repository**
   ```bash
   cd AW_GitBook
   git init
   git add .
   git commit -m "Initial GitBook framework"
   git remote add origin <your-repo-url>
   git push -u origin main
   ```

2. **Connect to GitBook**
   - Log in to GitBook (https://www.gitbook.com)
   - Click "New Space"
   - Select "Import from Git"
   - Choose your repository
   - GitBook will automatically detect `.gitbook.yaml`

3. **Configure Settings**
   - Set visibility (public/private)
   - Configure custom domain (optional)
   - Enable search indexing
   - Set up access permissions

4. **Publish**
   - GitBook will automatically build and deploy
   - Changes pushed to Git will auto-sync

### Option 2: Static Site Generation

#### Using GitBook CLI (Legacy)
```bash
# Install GitBook CLI
npm install -g gitbook-cli

# Navigate to project
cd AW_GitBook

# Install dependencies
gitbook install

# Build static site
gitbook build

# Serve locally for testing
gitbook serve

# Deploy to web server
# Copy _book/ directory to your web server
```

#### Using MkDocs (Alternative)
```bash
# Install MkDocs
pip install mkdocs mkdocs-material

# Create mkdocs.yml configuration
# Convert SUMMARY.md to mkdocs navigation

# Build
mkdocs build

# Deploy
mkdocs gh-deploy  # For GitHub Pages
```

### Option 3: Custom Integration

#### For Documentation Portals
1. Parse Markdown files
2. Extract frontmatter metadata
3. Render with your preferred engine
4. Maintain navigation from SUMMARY.md

## AI Vector Store Integration

### Option 1: OpenAI Embeddings

```python
import openai
import yaml
import os

def process_markdown_file(filepath):
    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()
    
    # Split frontmatter and content
    if content.startswith('---'):
        parts = content.split('---', 2)
        frontmatter = yaml.safe_load(parts[1])
        body = parts[2].strip()
    else:
        frontmatter = {}
        body = content
    
    # Create embedding text
    embedding_text = f"""
    Title: {frontmatter.get('title', '')}
    Tags: {', '.join(frontmatter.get('tags', []))}
    Description: {frontmatter.get('long_description', '')}
    Content: {body}
    """
    
    # Generate embedding
    response = openai.Embedding.create(
        input=embedding_text,
        model="text-embedding-ada-002"
    )
    
    return {
        'filepath': filepath,
        'frontmatter': frontmatter,
        'content': body,
        'embedding': response['data'][0]['embedding']
    }

# Process all markdown files
for root, dirs, files in os.walk('AW_GitBook'):
    for file in files:
        if file.endswith('.md') and file not in ['README.md', 'SUMMARY.md']:
            filepath = os.path.join(root, file)
            result = process_markdown_file(filepath)
            # Store in your vector database
```

### Option 2: Pinecone Integration

```python
import pinecone
import openai
from pathlib import Path

# Initialize Pinecone
pinecone.init(api_key="your-api-key", environment="your-env")
index = pinecone.Index("aw-software-docs")

def index_document(filepath, embedding, metadata):
    # Create unique ID from filepath
    doc_id = filepath.replace('/', '_').replace('.md', '')
    
    # Upsert to Pinecone
    index.upsert([(doc_id, embedding, metadata)])

# Process and index all documents
for md_file in Path('AW_GitBook').rglob('*.md'):
    if md_file.name not in ['README.md', 'SUMMARY.md']:
        result = process_markdown_file(str(md_file))
        index_document(
            str(md_file),
            result['embedding'],
            result['frontmatter']
        )
```

### Option 3: Weaviate Integration

```python
import weaviate

# Connect to Weaviate
client = weaviate.Client("http://localhost:8080")

# Create schema
schema = {
    "classes": [{
        "class": "AWDocument",
        "description": "A+W Software documentation",
        "vectorizer": "text2vec-openai",
        "properties": [
            {"name": "title", "dataType": ["string"]},
            {"name": "source", "dataType": ["string"]},
            {"name": "tags", "dataType": ["string[]"]},
            {"name": "short_description", "dataType": ["text"]},
            {"name": "long_description", "dataType": ["text"]},
            {"name": "content", "dataType": ["text"]},
            {"name": "category", "dataType": ["string"]},
            {"name": "filepath", "dataType": ["string"]}
        ]
    }]
}

client.schema.create(schema)

# Index documents
def index_to_weaviate(result):
    client.data_object.create(
        data_object={
            "title": result['frontmatter'].get('title', ''),
            "source": result['frontmatter'].get('source', ''),
            "tags": result['frontmatter'].get('tags', []),
            "short_description": result['frontmatter'].get('short_description', ''),
            "long_description": result['frontmatter'].get('long_description', ''),
            "content": result['content'],
            "category": result['filepath'].split('/')[1],
            "filepath": result['filepath']
        },
        class_name="AWDocument"
    )
```

## Search and Retrieval

### Semantic Search Example

```python
def semantic_search(query, top_k=5):
    # Generate query embedding
    query_embedding = openai.Embedding.create(
        input=query,
        model="text-embedding-ada-002"
    )['data'][0]['embedding']
    
    # Search in Pinecone
    results = index.query(
        query_embedding,
        top_k=top_k,
        include_metadata=True
    )
    
    return results

# Example usage
results = semantic_search("How do I optimize glass cutting?")
for match in results['matches']:
    print(f"Title: {match['metadata']['title']}")
    print(f"Score: {match['score']}")
    print(f"Description: {match['metadata']['short_description']}\n")
```

### Tag-Based Filtering

```python
def search_by_tags(tags, query=None):
    filter_dict = {
        "tags": {"$in": tags}
    }
    
    if query:
        query_embedding = openai.Embedding.create(
            input=query,
            model="text-embedding-ada-002"
        )['data'][0]['embedding']
        
        results = index.query(
            query_embedding,
            top_k=10,
            filter=filter_dict,
            include_metadata=True
        )
    else:
        # Fetch all documents with these tags
        results = index.fetch(filter=filter_dict)
    
    return results

# Example usage
erp_docs = search_by_tags(["ERP"], "enterprise resource planning")
```

## Maintenance and Updates

### Updating Content

1. **Edit Markdown Files**
   - Update content as needed
   - Update `last_updated` in frontmatter
   - Increment `version` if significant changes

2. **Commit Changes**
   ```bash
   git add .
   git commit -m "Update product documentation"
   git push
   ```

3. **Re-index for Vector Store**
   ```python
   # Re-process and update embeddings
   updated_files = get_modified_files()  # Your implementation
   for file in updated_files:
       result = process_markdown_file(file)
       update_vector_store(result)  # Your implementation
   ```

### Adding New Documents

1. **Create Markdown File**
   - Place in appropriate category directory
   - Include complete frontmatter

2. **Update SUMMARY.md**
   - Add entry in correct location
   - Maintain hierarchy

3. **Commit and Deploy**
   ```bash
   git add .
   git commit -m "Add new product documentation"
   git push
   ```

4. **Index New Document**
   ```python
   result = process_markdown_file('path/to/new/file.md')
   index_document(result)
   ```

## Performance Optimization

### Chunking Strategy

For long documents, implement chunking:

```python
def chunk_document(content, chunk_size=1000, overlap=200):
    chunks = []
    start = 0
    while start < len(content):
        end = start + chunk_size
        chunk = content[start:end]
        chunks.append(chunk)
        start = end - overlap
    return chunks

def process_with_chunking(filepath):
    result = process_markdown_file(filepath)
    chunks = chunk_document(result['content'])
    
    embeddings = []
    for i, chunk in enumerate(chunks):
        embedding_text = f"{result['frontmatter']['title']} - Part {i+1}\n{chunk}"
        embedding = create_embedding(embedding_text)
        embeddings.append({
            'chunk_id': f"{filepath}_chunk_{i}",
            'content': chunk,
            'embedding': embedding,
            'metadata': result['frontmatter']
        })
    
    return embeddings
```

### Caching

Implement caching to avoid re-processing unchanged files:

```python
import hashlib
import json

def get_file_hash(filepath):
    with open(filepath, 'rb') as f:
        return hashlib.md5(f.read()).hexdigest()

def should_reprocess(filepath, cache_file='embedding_cache.json'):
    current_hash = get_file_hash(filepath)
    
    try:
        with open(cache_file, 'r') as f:
            cache = json.load(f)
        return cache.get(filepath) != current_hash
    except FileNotFoundError:
        return True

def update_cache(filepath, cache_file='embedding_cache.json'):
    current_hash = get_file_hash(filepath)
    
    try:
        with open(cache_file, 'r') as f:
            cache = json.load(f)
    except FileNotFoundError:
        cache = {}
    
    cache[filepath] = current_hash
    
    with open(cache_file, 'w') as f:
        json.dump(cache, f)
```

## Monitoring and Analytics

### Track Search Queries

```python
import logging

logging.basicConfig(filename='search_queries.log', level=logging.INFO)

def tracked_search(query):
    logging.info(f"Query: {query}")
    results = semantic_search(query)
    logging.info(f"Results: {len(results['matches'])}")
    return results
```

### Monitor Performance

```python
import time

def timed_search(query):
    start = time.time()
    results = semantic_search(query)
    duration = time.time() - start
    
    print(f"Search completed in {duration:.2f} seconds")
    return results
```

## Troubleshooting

### Common Issues

1. **Frontmatter Not Parsing**
   - Ensure YAML is valid
   - Check for proper `---` delimiters
   - Verify UTF-8 encoding

2. **GitBook Not Building**
   - Validate `.gitbook.yaml` syntax
   - Check SUMMARY.md links
   - Ensure all referenced files exist

3. **Poor Search Results**
   - Review embedding quality
   - Adjust chunk sizes
   - Enhance frontmatter descriptions
   - Add more relevant tags

4. **Slow Indexing**
   - Implement caching
   - Use batch processing
   - Optimize chunk sizes

## Best Practices

1. **Version Control**: Always use Git for tracking changes
2. **Backup**: Regularly backup vector store indices
3. **Testing**: Test search queries before production deployment
4. **Documentation**: Keep this guide updated with changes
5. **Monitoring**: Track usage and performance metrics

## Support Resources

- GitBook Documentation: https://docs.gitbook.com
- OpenAI API: https://platform.openai.com/docs
- Pinecone Documentation: https://docs.pinecone.io
- Weaviate Documentation: https://weaviate.io/developers/weaviate

---

**Last Updated**: 2025-10-28  
**Version**: 1.0
