Vacuum documents into an Elasticsearch AI assistant knowledgebase
=========================================================

This project is to simplify the creation of knowledgebase indices for the AI assistants
in the Elastic Stack. Currently supported input files:

 - PDF

The two example used throughout this document will be:

 **(A)** The "IT-Grundschutz-Kompendium – Werkzeug für Informationssicherheit" from the German BSI.
 An 800+ page PDF of IT guidelines; [https://www.bsi.bund.de/DE/Themen/Unternehmen-und-Organisationen/Standards-und-Zertifizierung/IT-Grundschutz/IT-Grundschutz-Kompendium/it-grundschutz-kompendium_node.html]

 **(B)** The "IT-Grundschutz-Bausteine" - Broken down to multiple PDFs files in one zip download
 [https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/Zip_Datei_Edition_2023.html]

Prerequisits
============

 - The Elastic Stack (In cloud or on-prem) with at least 8GB of ML node capacity available
   - This script will try to deploy elastics e5 model [https://www.elastic.co/guide/en/machine-learning/current/ml-nlp-e5.html]
 - MacOS or Linux to run the bash script with the following tools installed
   - `curl jq pdftk sed awk grep cut openssl`
 - One or more PDF files you'd like to ask questions about or be used to correctly guide your analysts

Installing needed shell tools
-----------------------------

### Linux

Use your OS package manager. e.g. `apt-get curl jq pdftk` (The others should already be installed).
It is assumed Linux users know how to install such common shell packages.

### MacOS

I like to use Nix as a package manager [https://nixos.org/download/]. Therefore I can for example do `nix-env -i pdftk` to add pdftk to my user's shell environment.

Many other MacOS users like to use Brew [https://brew.sh/]

### Windows

Although untested, its noted that Nix supports WSL2, so you could try Nix [https://nixos.org/download/] there too.

----

Usage
=====

The main script is eskb_doc_vac, which is a bash script and should be made executable.
Its run as `./eskb_doc_vac <command>` we will review these commands in order of usage:-

help
----

Lists all available commands and if a command-name is additionally given it will show more infomation on that command

e.g. (A) & (B)

```
./eskb_doc_vac help
./eskb_doc_vac help config
```

config
------

Configure your connection to Elasticsearch. This will generate a config file in the current directory, to be used by all other commands.

e.g. (A) & (B)
```
./eskb_doc_vac config https://es-url.example.com some_username some_password
```

The three argument are optional, but if omitted you will need to edit the resulting file
manually to set the URL, Username and Password. Other advanced settings can also be edited in the config file.

Regarding examples (A) and (B), these are to highlight two possible excerpt strategies.
For (A) strategy `bookmark1` and for (B) strategy `whole`.

The default (set by running `config`) is `auto` - Which automatically switches between whole or bookmark1
depending on the input file size. (The threashold for which is set at the top of the script).

This means for the two examples (A) & (B), you do not need to edit the config file.
However if you want to set an explicit strategy, you can edit the file and change the strategy.
For example to `whole`:
```
...
#strategy "whole"
...
```

See read_pdf below for more infomation on this


ping
----

Check that your Elasticsearch configuration is correct, and that you can access the instance.

e.g. (A) & (B)
```
./eskb_doc_vac ping
```

setup_all
---------

Performs all the follwing setup steps in one go.

e.g. (A) - Spliting a large PDF by bookmark level1 sections
```
./eskb_doc_vac setup_all kb_bsi_compendium
```

e.g. (B) - Ingesting whole small PDFs
```
./eskb_doc_vac setup_all kb_bsi_collection
```

or call each separately:

### setup_inference

This will create the inference instance, possibly triggering a model download if needed.

### setup_ingest

This will create the ingest pipeline used to process all incoming excerpt documents.

### setup_index

This will create an index to use as a knowledgebase, with all the required settings and mappings. It takes one argument, the name of the index.

read_pdf
--------

Read a PDF file (name give in second argument) into the knowledgebase index (name given in first argument).

The file will be read in one excerpt at a time, each excerpt becoming its own Elasticsearch document.
There are four excerpt strategies:

 - auto - Switch automatically between 'whole' or 'bookmark1', depending on input file size (default)
 - whole - Excerpt is the whole document
 - bookmark1 - Excerpt are all pages between Level 1 Bookmarks
 - page - Excerpts are each page in turn

 The bookmark1 strategy follows a document structure splitting approach that would give best results if
 documents are too big to ingest whole. Whole documents would give best results if they are small enough.
 (edit the config file to change the strategy)

e.g. (A) - Strategy configured to `bookmark1` or `auto` (as the file is over 0.5M, it's bigger than 5M)
```
./eskb_doc_vac read_pdf kb_bsi_compendium ~/Downloads/IT_Grundschutz_Kompendium_Edition2023.pdf
```

e.g. (B) - Strategy configured to `whole` or `auto` (as no file is bigger than 0.5M)
```
unzip ~/Downloads/Zip_Datei_Edition_2023.zip
./eskb_doc_vac read_pdf kb_bsi_collection ./Einzeln_PDF/*.pdf
```

That's it! Once read in you can add the index as a knowledgebase to either assistant (Obervability or Security), alternatively you can play with it in the playground.

Playground: In kibana from the hamburger button navigate to Search -> Playground. There select your LLM connector and the index you created with this script (e.g. `kb_bsi`).


---

Under the hood
==============

ES document schema
------------------

 - excerpt:
   - content - The actual excerpt text
   - title - The title for excerpt
   - bookmarks - A list of bookmarks contained within the excerpt
   - start_page - The document page where this excerpt starts
   - end_page - The document page where this excerpt ends
 - document:
   - format - Currently just PDF
   - filename - Base filename
   - strategy - Excerpt strategy (whole, page, or bookmark1)
 - semantic_content - Semantic text search field

The index schema is created in do_setup_index() of the script

ES ingest pipeline
------------------

Expects the following original doc input fields:
 - _data - Base64 encoded pdf excerpt (binary) content, from which text will be extracted
 - _from - Start page number
 - _to - End page page number
 - _title - Excerpt title
 - _bookmarks - Excerpt bookmark list
 - document.*

The pipeline will take _fields and create the excerpt.* object.

This pipeline is created in do_setup_ingest() of the script


Inside the script
-----------------

The script eskb_doc_vac is inteded to be fairly self documenting, if you can read and use
curl commands, you can read the script and follow along. Each command is a function called
'do_`command`' and none are that long or complicated, bar the embedded json documents.

----

Changes / Updates
=================

The original script was built around the new Elastic e5 multilingual model. However its dense_vectors*
are not currently supported by the Elastic Security AI Assistant (my main interest).
So this script has been changed to use "ELSER" (English trained model), which gives sparse_vectors*.

It was still able to converse with me regarding the German language BSI docs used in the examples here.

The original e5 script can be found in ./file_vac/ and/or you can change the EMBEDDING_MODE var in the new script.

(*) The meaning of sparse vs. dense vectors is not important here, just that they are not the same thing.

Acknowledgements
================

My collegeue Christine Kommander wrote this great blog post [https://www.elastic.co/search-labs/blog/rag-with-pdfs-genai-search] that I also referenced.


Licence, Author & Copyright
===========================

Copyright: 2024 Elasticsearch N.V.

Author: Thorben Jändling

Licence: LGPL v2.1 or any minor 2.x version there above.
