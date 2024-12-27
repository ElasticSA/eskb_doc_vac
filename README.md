Elasticsearch AI assistant knowledgebase creator (vacuum)
=========================================================

This project is to simplify the creation of knowledgebase indices for the AI assistants
in the Elastic Stack. Currently supported input files:

 - PDF

The example used throughout this document will be the "IT-Grundschutz-Kompendium – Werkzeug für Informationssicherheit" from the German BSI. An 800+ page PDF of IT guidelines.

Prerequisits
============

 - The Elastic Stack (In cloud or on-prem) with at least 8GB of ML node capacity available
 - Install and deploy the E5 model [TODO Link]
 - MacOS or Linux to run the bash script with the following tools installed
   - curl, jq, pdftk, sed, grep
 - One or more PDF files you'd like to ask questions about or be used to correctly guide your analysts

Usage
=====

The main script is eskb_doc_vac, which is a bash script and should be made executable.
Its run as `./eskb_doc_vac <command>` we will review these commands in order of usage:-

help
----

Lists all available commands and if a command is additionally give it will show more infomation on that command

e.g.

```
./eskb_doc_vac help
```

config
------

Configure your connection to Elasticsearch. This will generate a config file in the current directory, to be used by all other commands.

e.g.
```
./eskb_doc_vac config https://es-url.example.com some_username some_password
```

The three argument are optional, but if omitted you will need to edit the resulting file
manually to set the URL, Username and Password.

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
./eskb_doc_vac setup_all kb_bsi_grundschutz
```

setup_inference
---------------

Assuming you have the E5 small model deployed, this will create the inference instance.

setup_ingest
------------

This will create the ingest pipeline used to process all incoming page documents.

setup_index
-----------

This will create an index to use as a knowledgebase, with all the required settings and mappings. It takes one argument, the name of the index.

read_pdf
--------

Read a PDF file (name give in second argument) into the knowledgebase index
(name given in first argument). The file will be read in one page at a time, each page becoming its own Elasticsearch document.

e.g.
```
./eskb_doc_vac read_pdf kb_bsi_grundschutz ~/Downloads/IT_Grundschutz_Kompendium_Edition2023.pdf
```

That's it! Once read in you can add the index as a knowledgebase to either assistants, or play with it in the playground (TODO more detail guidance)

Under the hood
==============

The script eskb_doc_vac is inteded to be fairly self documenting, if you can read and use
curl commands, you can read the script and follow along. Each command is a function called
'do_`command`' and none are that long or complicated, bar the embeeded json documents.

Acknowledgements
================

My collegeue Christine Kommander wrote this great blog post [https://www.elastic.co/search-labs/blog/rag-with-pdfs-genai-search] that I also referenced.


Licence, Author & Copyright
===========================

Copyright: 2024 Elasticsearch N.V.

Author: Thorben Jändling

Licence: LGPL v2.1 or any minor 2.x version there above.
