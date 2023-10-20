
```bash
brew install Caskroom/cask/wkhtmltopdf
pip install pdfkit
pip install PdfMerger
pip install PDFNetPython3 
```

Fetch all pages and turn them into one big pdf file

```python
import os
from PyPDF2 import PdfMerger
import pdfkit
from urllib.parse import urlparse

# Use standard library to fetch contents of URLs
import requests
# Use standard library to parse fetched sitemaps
import xml.dom.minidom as minidom

# Fetch sitemap's text of each site that is to be indexed
fermyon_website_sitemap = requests.get('https://www.fermyon.com/sitemap.xml', allow_redirects=True).text
fermyon_documentation_sitemap = requests.get('https://developer.fermyon.com/sitemap.xml', allow_redirects=True).text
component_model_documentation_sitemap = requests.get('https://component-model.bytecodealliance.org/sitemap.xml', allow_redirects=True).text

# Parse each sitemap's text to obtain list of pages
parsed_fermyon_website_sitemap_document = minidom.parseString(fermyon_website_sitemap)
parsed_fermyon_documentation_sitemap_document = minidom.parseString(fermyon_documentation_sitemap)
parsed_component_model_documentation_sitemap_document = minidom.parseString(component_model_documentation_sitemap)

# Cherry pick just the loc elements from the XML
fermyon_website_sitemap_loc_elements = parsed_fermyon_website_sitemap_document.getElementsByTagName('loc')
fermyon_documentation_sitemap_loc_elements = parsed_fermyon_documentation_sitemap_document.getElementsByTagName('loc')
component_model_documentation_sitemap_loc_elements = parsed_component_model_documentation_sitemap_document.getElementsByTagName('loc')

# Declare blank lists of pages for each site
fermyon_website_page_urls = []
fermyon_documentation_page_urls = []
component_model_documentation_page_urls = []

# Iterate over loc elements (of each sitemap) and add to that site's list of pages
for fermyon_website_sitemap_loc_element in fermyon_website_sitemap_loc_elements:
    fermyon_website_page_urls.append(fermyon_website_sitemap_loc_element.toxml().removesuffix("</loc>").removeprefix("<loc>"))
for fermyon_documentation_sitemap_loc_element in fermyon_documentation_sitemap_loc_elements:
    fermyon_documentation_page_urls.append(fermyon_documentation_sitemap_loc_element.toxml().removesuffix("</loc>").removeprefix("<loc>"))
for component_model_documentation_sitemap_loc_element in component_model_documentation_sitemap_loc_elements:
    component_model_documentation_page_urls.append(component_model_documentation_sitemap_loc_element.toxml().removesuffix("</loc>").removeprefix("<loc>"))

URLs = fermyon_website_page_urls + fermyon_documentation_page_urls + component_model_documentation_page_urls

print("Number of page to process is {}\n First page to process is {} and the last page to process is {}".format(len(URLs), URLs[0], URLs[len(URLs) - 1]))

text_to_remove = "rss"
filtered_list = [item for item in URLs if text_to_remove not in item]

for item in filtered_list:
    temp = urlparse(str(item))
    pdf_file_name = temp.netloc + temp.path + '.pdf'
    pdf_file_name_clean = pdf_file_name.replace("/", "-")
    pdfkit.from_url(str(item), pdf_file_name_clean, verbose=True)

x = [a for a in os.listdir() if a.endswith(".pdf")]

merger = PdfMerger()

for pdf in x:
    merger.append(open(pdf, 'rb'))

with open("result.pdf", "wb") as fout:
    merger.write(fout)
```

Compress the master PDF into a smaller one to upload to openchat.
Save the following code to a file called `pdf_compressor.py`

```python3
import os
import sys
from PDFNetPython3.PDFNetPython import PDFDoc, Optimizer, SDFDoc, PDFNet

def get_size_format(b, factor=1024, suffix="B"):
    """
    Scale bytes to its proper byte format
    e.g:
        1253656 => '1.20MB'
        1253656678 => '1.17GB'
    """
    for unit in ["", "K", "M", "G", "T", "P", "E", "Z"]:
        if b < factor:
            return f"{b:.2f}{unit}{suffix}"
        b /= factor
    return f"{b:.2f}Y{suffix}"

def compress_file(input_file: str, output_file: str):
    """Compress PDF file"""
    if not output_file:
        output_file = input_file
    initial_size = os.path.getsize(input_file)
    try:
        # Initialize the library
        PDFNet.Initialize()
        doc = PDFDoc(input_file)
        # Optimize PDF with the default settings
        doc.InitSecurityHandler()
        # Reduce PDF size by removing redundant information and compressing data streams
        Optimizer.Optimize(doc)
        doc.Save(output_file, SDFDoc.e_linearized)
        doc.Close()
    except Exception as e:
        print("Error compress_file=", e)
        doc.Close()
        return False
    compressed_size = os.path.getsize(output_file)
    ratio = 1 - (compressed_size / initial_size)
    summary = {
        "Input File": input_file, "Initial Size": get_size_format(initial_size),
        "Output File": output_file, f"Compressed Size": get_size_format(compressed_size),
        "Compression Ratio": "{0:.3%}.".format(ratio)
    }
    # Printing Summary
    print("## Summary ########################################################")
    print("\n".join("{}:{}".format(i, j) for i, j in summary.items()))
    print("###################################################################")
    return True

if __name__ == "__main__":
    # Parsing command line arguments entered by user
    input_file = sys.argv[1]
    output_file = sys.argv[2]
    compress_file(input_file, output_file)
```

```bash
python3 pdf_compressor.py result.pdf result_compressed.pdf
```
