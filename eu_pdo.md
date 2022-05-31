PDO
================
Tuija Sonkkila

# <https://rmarkdown.rstudio.com/github_document_format.html>

``` r
library(reticulate)
```

## Python setup

Followed advice from
[here](https://rpubs.com/keithmcnulty/r_and_python).

Start Anaconda Prompt; create and activate an environment; install
libraries with conda or pip; copy active environment location, append it
with **\\bin\\python3** and store in .Renviron as the value of the
RETICULATE_PYTHON environment variable.

Note double backslashes because Windows.

    conda create --name r_and_python
    conda activate r_and_python

    conda install [lib]

    conda install pip
    pip install [lib])

You cannot run individual Python chunks. Knit everything.

``` python
import os
from datetime import datetime
import pandas as pd
import json
from SPARQLWrapper import SPARQLWrapper, JSON, POST
import zipfile
import requests
import io

timestamp = str(datetime.now().strftime("%Y%m%d-%H%M%S"))
```

``` python
def text_to_str(file_path):
    """
    Read lines of file in given path
    and return string of the text.
    :param file_path: file path str
    :return: text str
    """
    with open(file_path, 'r') as file:
        return file.read()
```

``` python
def get_cellar_info_from_endpoint(sparql_query):
    """
    Send the given sparql_query to the EU Sparql endpoint
    and retrieve and return the results in JSON format.

    :param sparql_query: str
    :return: json dict
    """
    endpoint = "http://publications.europa.eu/webapi/rdf/sparql" # 2020-06-12 THIS
    sparql = SPARQLWrapper(endpoint)
    sparql.setQuery(sparql_query)
    sparql.setMethod(POST)
    sparql.setReturnFormat(JSON)
    results = sparql.query().convert()
    return results
```

``` python
def to_json_output_file(file_name, data):
    """
    Print the given data input to 
    a file in json format with the given file_name.
    :param file_name: str
    :param data: structured data
    :return: text file
    """
    with open(file_name, 'w') as outfile:
        json.dump(data, outfile, indent=4)
        
```

``` python
def query_results_to_json(query_results):
    """
    Output query results to json file.
    :param query_results: dict
    :return: None
    """
    to_json_output_file('res_'+timestamp+'.json', query_results)
```

``` python
def get_cellar_ids_from_json_results(cellar_results):
    """
    Create a list of CELLAR ids from the given cellar_results JSON dictionary and return the list.

    :param cellar_results: dict
    :return: list of cellar ids
    """
    results_list = cellar_results["results"]["bindings"]
    # List comprehension / ttso
    cellar_ids_list = [results_list[i]["cellarURIs"]["value"].split('/')[-1] for i in range(len(results_list))]
    return cellar_ids_list
```

``` python
def rest_get_call(id):
    """Send a GET request to download a zip file for the given id under the CELLAR URI."""
    url = 'http://publications.europa.eu/resource/cellar/' + id
    
    headers = {
        'Accept': "application/zip;mtype=fmx4, application/xml;mtype=fmx4, application/xhtml+xml, text/html, text/html;type=simplified, application/msword, text/plain, application/xml;notice=object",
        'Accept-Language': "eng",
        'Content-Type': "application/x-www-form-urlencoded",
        'Host': "publications.europa.eu"#,
    }
    response = requests.request("GET", url, headers=headers)
    return response
```

``` python
def download_zip(response, folder_path):
    """
    Downloads the zip file returned by the restful get request.
    Source: https://stackoverflow.com/questions/9419162/download-returned-zip-file-from-url?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa
    """
    z = zipfile.ZipFile(io.BytesIO(response.content))
    z.extractall(folder_path)
    
```

``` python
def process_range(sub_list, folder_path):
    """
    Process a list of ids to download the corresponding zip files.
    :param sub_list: list of str
    :param folder_path: str
    :return: write to files
    """

    # Keep track of downloads
    zip_files = []
    single_files = []
    other_downloads = []

    # Count downloads
    count_cellar_ids = 0
    count_zip = 0
    count_single= 0
    count_other = 0

    for id in sub_list:
        count_cellar_ids += 1

        # Specify sub_folder_path to send results of request
        sub_folder_path = folder_path + id

        # Send Restful GET request for the given id
        response = rest_get_call(id.strip())

        # If the response's header contains the string 'Content-Type'
        if 'Content-Type' in response.headers:

            # If the string 'zip' appears as a value of 'Content-Type'
            if 'zip' in response.headers['Content-Type']:

                count_zip += 1
                zip_files.append(id)

                # Download the contents of the zip file in the given folder
                download_zip(response, sub_folder_path)

            # If the value of 'Content-Type' is not 'zip'
            else:
                count_single += 1
                single_files.append(id)

                # Create a directory with the cellar_id name
                # and write the returned content in a file
                # with the same name
                out_file = sub_folder_path + '/' + id + '.html'
                os.makedirs(os.path.dirname(out_file), exist_ok=True)
                with open(out_file, 'w') as f:
                    f.write(response.text)

        # If the response's header does not contain the string 'Content-Type'
        else:
            count_other += 1
            other_downloads.append(id)
           
    # Write the list of other (failed) downloads in a file
    id_logs_path = 'id_logs/failed_' + timestamp + '.txt'
    os.makedirs(os.path.dirname(id_logs_path), exist_ok=True)
    with open(id_logs_path, 'w+') as f:
        if len(other_downloads) != 0:
            f.write('Failed downloads ' + timestamp + '\n' + str(other_downloads))
```

``` python
sparql_query = text_to_str('pdo.rq')
```

``` python
sparql_query_results = get_cellar_info_from_endpoint(sparql_query)
```

``` python
query_results_to_json(sparql_query_results)
```

``` python
id_list = get_cellar_ids_from_json_results(sparql_query_results)
print('ID LIST:', len(id_list), id_list)
```

    ## ID LIST: 5 ['edb9e042-1a93-486c-85b9-2c15339adc66', 'ac3da261-07b0-4247-9b0c-010d076f6267', 'f715ce02-8368-40d0-93ff-e724c18d9362', '29861b38-02db-46a4-a09d-28c983776a43', 'ed3b528e-2170-40a1-8462-78a91a624491']

``` python
dwnld_folder_path = "data/cellar_files_" + timestamp + "/"
#txt_folder_path = "data/text_files_" + dwnld_folder_path.split('_')[-1]

process_range(id_list, dwnld_folder_path)
```

\#data/cellar_files\_\[timestamp\]/\[cellarID\]/*.xml (not *.doc.xml)
\#CONSID/NP/NOTE/P/REF.DOC.OJ\[@COLL="C"\] -\> OJ C 260, 9.8.2014, p. 24
\#@DATE.PUB=“20140809” \#@NO.OJ=“260” \#@PAGE.FIRST=“24”
