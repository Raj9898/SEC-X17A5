# X-17A-5 Optical Character Recognition (OCR)

## 1	Introduction
The project scrapes the SEC for X-17A-5 filings published by registered broker-dealers historically, and constructs a database for asset, liability and equity line items. The project runs on Amazon Web Services (AWS) in a SageMaker instance and stores the scraped information into an s3 bucket. 

## 2	Software Dependencies
**All code is executed using Python 3.6 as of current release. We make no claim for stability on other version of Python.**

We use [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) to interact with the SEC website and EDGAR archive, to extract data files (e.g. X-17A-5). 
```
pip install bs4
```

We use [PyPDF2](https://pythonhosted.org/PyPDF2/), [PyMuPDF](https://github.com/pymupdf/PyMuPDF), [pdf2image](https://pypi.org/project/pdf2image/), [fitz](https://pypi.org/project/fitz/), and [pillow](https://pillow.readthedocs.io/en/stable/) as used in retrieval & slicing operations. 
```
pip install PyPDF2
pip install PyMuPDF
pip install pdf2image 
pip install pillow
pip install fitz 
pip install cv2     
```

PLEASE READ THE DOCUMENTATION FROM pdf2image provided at the following [link](https://github.com/Belval/pdf2image). You will need to install poppler on your machine (e.g. Windows, Mac, Linux) to execute slicing operations. For additional details on poppler see [doc](https://poppler.freedesktop.org/) on use cases. 
```
# to install poppler PDF backend for analysis on Jupyter environment  
conda install -c conda-forge poppler  -y
```

We use [smart_open](https://pypi.org/project/smart-open/), [minecart](https://pypi.org/project/minecart/), and [textract-trp](https://pypi.org/project/textract-trp/) to analyze X-17A-5 filings using AWS Textract to perform OCR on both PDFs and PNGs.   
```
pip install smart_open
pip install minecart
pip install textract-trp
```

We use [python-Levenshtein](https://pypi.org/project/python-Levenshtein/), and [fuzzywuzzy](https://pypi.org/project/fuzzywuzzy/) for performing "fuzzy" string matching for particular strings
```
pip install python-Levenshtein
pip install fuzzywuzzy
```

All Python installations are handled via a python `subprocess` from our `run_main.py` script, which executes each installation via the following:

```
subprocess.check_call([sys.executable, '-m', 'pip', 'install', 'bs4'])
```

## 3	File Structure

### 3.1 	Resource Files

* `CIKandDealers.json` JSON file storing CIK numbers for firms and company names as key/value pairs respectively ({"broker-dealers" : {"356628": "NATIONAL FINANCIAL SERVICES LLC", "815855": "MERRILL LYNCH GOVERNMENT SECURITIES OF PUERTO RICO INC"}}), with accompanying years covered {'years-covered': ["1993/QTR1", "1993/QTR2", "1993/QTR3", "1993/QTR4"]}. All CIK numbers are taken from the EDGAR [archive](https://www.sec.gov/Archives/edgar/full-index/) from the SEC. 

* `X17A5-FORMS.json` JSON file storing the CIK numbers with the accompanying [FORMS](https://docs.aws.amazon.com/textract/latest/dg/how-it-works-kvp.html) data retrieved from AWS Textract.

* `X17A5-TEXT.json` JSON file storing the CIK numbers with the accompanying [TEXT](https://docs.aws.amazon.com/textract/latest/dg/how-it-works-lines-words.html) data retrieved from AWS Textract.

### 3.2 	Error Files

* `ERROR-TEXTRACT.json` JSON file storing CIK numbers with accompanying year that were unable to be read via Textract. There are two types of errors that are raised:
    * *No Balance Sheet found, or parsing error*, where there may be an issue with Textract reading the page
    * *Could not parse, JOB FAILED*, where there may be an issue with Textract parsing the pdf file   
    
### 3.3 	Input Files

* `asset_log_reg_mdl_v2.joblib` & `liability_log_reg_mdl_v2.joblib` logistic regression classification models used for constructing the structured database, for asset and liablity & equity line items respectively
    
### 3.4 	Code Files

The code files are divided into three sub-groups that are responsible for executing a portion of the database construction and are found within the `/code/src` folder.  

#### 3.4a 

   * `GLOBAL.py` Script storing essential global variables regarding mapping conventions for s3 directories
   
   * `run_main.py` the primary scipt for execution of all code-steps Part 1, Part 2 and Part 3 (see below 3.3b) 

   * `run_file_extraction.py` runs all execution for Part 1 (see below 3.3b), responsible for gathering FOCUS reports and building list of broker-dealers 

   * `run_ocr.py` runs all execution for Part 2 (see below 3.3b), responsible for extracting balance-sheet figures by OCR via AWS Textract
   
   * `run_ocr.py` runs all execution for Part 3 (see below 3.3b), responsible for developing structured and unstructured database

#### 3.3b 	

##### Part 1: Broker-Dealer and FOCUS Report Extraction

   * `ExtractBrokerDealers.py` responsible for creating the `CIKandDealers.json` file, which stores all CIK-Name information for broker-dealers that file an X-17A-5.   
   * `FocusReportExtract.py` responsible for extracting the X-17A-5 pdf files from broker-dealer URLs
   * `FocusReportSlicing.py` reduces the size of the X-17A-5 pdf files to a "manageable" subset of pages with corresponding PNG(s)

##### Part 2: Optical Character Recognition

   * `OCRTextract.py` calls the AWS asynchronous Textract API to perform OCR on the reduced X-17A-5 filings, selecting only the balance sheet and uploading it to a s3 bucket
   * `OCRClean.py` refines the scraped balance sheet data from Textract, handling case exemptions such as merged rows, multiple columns and numeric string conversions 

##### Par 3: Database construction

   * `DatabaseSplits.py` divides the balance sheet into asset terms and liability & equity terms for pdf(s) and png(s)
   * `DatabaseUnstructured.py` constructs a semi-finished database that captures all unique line items by column for the entirety of each bank and year
   * `DatabaseStructured.py` constructs a finished database that aggregates columns by a predicted class for assets and liability & equity terms

### 3.5 	Output Files

   * `unstructured_assets.csv` & `unstructured_liable.csv` represent the asset and liability & equity balance sheets respectively, non-aggregated by line items for each broker-dealers per filing year - where our values correspond to parsed FOCUS reports.   

   * `asset_name_map.csv` & `liability_name_map.csv` represent the asset and liability & equity line item mappings, as detrmined by our logistic regression classifier model - mapping balance-sheet line items to one of our pre-defined accounting groups.   

   * `structured_assets.csv` & `structured_liability.csv` represent the asset and liability & equity balance sheets respectively, aggregating by line items for each broker-dealers per filing year - where aggregation is determined by a logistic regression classifier model and values correspond to parsed FOCUS reports.  

## 4	Running Code

Our code file runs linearly in a SageMaker instance via batch. 

1. TBD

## 5	Possible Extensions
* Streamline the runtime of each file via batch job on AWS
* Apply parrell computing capabilites to Textract for running multiple jobs
* Extend and modify idiosyncratic changes as deemed appropriate for when Textract fails

## 6	Contributors
* [Rajesh Rao](https://github.com/Raj9898) (Sr. Research Analyst 22’)
* [Fernando Duarte](https://github.com/fernando-duarte) (Sr. Economist)
