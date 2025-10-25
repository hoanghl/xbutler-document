# System design

version: v1.1

# Features and requirements

1. Upload resource
1. Search image given the patch of image (both in raw image and information of bounding box)
1. Search image given the information of image/video already in the database
1. Integrate with Object Store

# Description

Embedding extractor:

- Extract image embedding
- Extract keyword embedding

LLM:

- Extract keywords in specified categories

Attribute DB:

- Store image metadata

Embedding DB:

- Store image embedding
- Store word embedding
- Search similar images given embedding
- Search similar keywords given embedding
