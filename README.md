Vacuum documents into an Elasticsearch AI assistant knowledgebase
=========================================================

This project is to simplify the creation of knowledgebase indices for the AI assistants
in the Elastic Stack. Currently supported input files:

 - PDF

The example used throughout this document will be the "IT-Grundschutz-Kompendium – Werkzeug für Informationssicherheit" from the German BSI. An 800+ page PDF of IT guidelines; or the multi-file
zip download

Prerequisits
============

 - The Elastic Stack (In cloud or on-prem) with at least 8GB of ML node capacity available
   - This script will try to deploy elastics e5 model [https://www.elastic.co/guide/en/machine-learning/current/ml-nlp-e5.html]
 - MacOS or Linux to run the bash script with the following tools installed
   - curl, jq, pdftk, sed, grep, cut, etc...
 - One or more PDF files you'd like to ask questions about or be used to correctly guide your analysts

----

Usage
=====

The main script is eskb_doc_vac, which is a bash script and should be made executable.
Its run as `./eskb_doc_vac <command>` we will review these commands in order of usage:-

help
----

Lists all available commands and if a command-name is additionally given it will show more infomation on that command

e.g.

```
./eskb_doc_vac help
./eskb_doc_vac help config
```

config
------

Configure your connection to Elasticsearch. This will generate a config file in the current directory, to be used by all other commands.

e.g.
```
./eskb_doc_vac config https://es-url.example.com some_username some_password
```

The three argument are optional, but if omitted you will need to edit the resulting file
manually to set the URL, Username and Password. Other advanced settings can also be edited in the config file.

ping
----

Check that your Elasticsearch configuration is correct, and that you can access the instance.

e.g.
```
./eskb_doc_vac ping
```

setup_all
---------

Performs all the follwing setup steps in one go.

e.g.
```
./eskb_doc_vac setup_all kb_bsi
```

setup_inference
---------------

This will create the inference instance, possibly triggering a model download if needed.

setup_ingest
------------

This will create the ingest pipeline used to process all incoming excerpt documents.

setup_index
-----------

This will create an index to use as a knowledgebase, with all the required settings and mappings. It takes one argument, the name of the index.

read_pdf
--------

Read a PDF file (name give in second argument) into the knowledgebase index (name given in first argument).

The file will be read in one excerpt at a time, each excerpt becoming its own Elasticsearch document.
There are three excerpt strategies:

 - whole - Excerpt is the whole document
 - page - Excerpts are each page in tern
 - bookmark1 - Excerpt are all pages between Level 1 Bookmarks (default)

 The bookmark1 strategy follows a document structure splitting approach that would give best results if
 documents are too big to ingest whole. Whole documents would give best results if they are small enough.
 (edit the config file to change the strategy)

e.g.
```
./eskb_doc_vac read_pdf kb_bsi ~/Downloads/IT_Grundschutz_Kompendium_Edition2023.pdf
```

That's it! Once read in you can add the index as a knowledgebase to either assistant (Obervability or Security), alternatively you can play with it in the playground.

Playground: In kibana from the hamburger button navigate to Search -> Playground. There select your LLM connector and the index you created with this script (e.g. `kb_bsi`).

---

Under the hood
==============

ES document schema
------------------

 - excerpt:
   - content - The actual page text
   - title - The title for excerpt
   - bookmarks - A list of bookmarks contained within the excerpt
   - start_page - The document page where this excerpt starts
   - end_page - The document page where this excerpt ends
 - document:
   - format - Currently just PDF
   - filename - Base filename
   - strategy - Excerpt strategy (whole, page, or bookmark1)
 - content - Semantic text search field

The index schema is created in do_setup_index() of the script

ES ingest pipeline
------------------

Expects the following original doc input fields:
 - _data - Base64 encoded pdf page (binary) content, from which text will be extracted
 - _from - Start page number
 - _to - End page page number
 - _title - Excerpt title
 - _bookmarks - Excerpt bookmark list
 - document.*

The pipeline will take _fields and create the excerpt.* object.
It will then populate the 'content' semantic search field.

This pipeline is created in do_setup_ingest() of the script


Inside the script
-----------------

The script eskb_doc_vac is inteded to be fairly self documenting, if you can read and use
curl commands, you can read the script and follow along. Each command is a function called
'do_`command`' and none are that long or complicated, bar the embedded json documents.

----

Acknowledgements
================

My collegeue Christine Kommander wrote this great blog post [https://www.elastic.co/search-labs/blog/rag-with-pdfs-genai-search] that I also referenced.


Licence, Author & Copyright
===========================

Copyright: 2024 Elasticsearch N.V.

Author: Thorben Jändling

Licence: LGPL v2.1 or any minor 2.x version there above.
