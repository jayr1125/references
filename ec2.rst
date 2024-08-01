eCrash Project Documentation
============================

eCrash Input-Output
-------------------

**Input**: TIFF image

**Output**: JSON containing the relevant fields


eCrash Process Flow
-------------------

1. **Reader Utility**
   - Contains the watch folder application and monitors for TIFF images.
   - A reader utility service sends a file request via an API. The file manager service responds with the state name and the filename from its repository (assumed to be tied to a repository or directory of images).

2. **Form Type Checker**
   - Checks the input document to determine if it is a crash or non-crash document and whether the form is handwritten or not.
   - If handwritten, it is pushed to manual keying via an API call.

3. **Processing Typewritten Forms**
   - Sent to both the ML bucket/container and the OCR container for processing.

4. **Image Preprocessor**
   - Performs resizing and enhancements to the TIFF image.
   - The ML model performs data point detection and labeling.

5. **Preprocess Checker**
   - Checks if the input document needs a deep cleaning process such as pipe and noise removal.

6. **OCR Process**
   - Converts the TIFF image to a searchable PDF using OmniPage.

7. **Text Extractor**
   - Extracts text information based on data point detection.
   - Performs post-processing like name and address segmentation and grouping of vehicle and party information.

8. **Storing Extracted Fields**
   - Extracted fields are stored in JSON format and mapped.

9. **Sending JSON Response**
   - JSON file is sent as a response when the keying process service sends an API request.


API Listener
------------

1. Interacts with the API to get a list of files to be processed.
2. Loops each file and moves it to the ML and OCR process.


Content Extraction
------------------

1. Script monitors ML and OCR outputs.
   - Converts TIFF images to .csv and .pdf respectively using a Python script corresponding to bounding box coordinates and a searchable PDF.

2. Reads .csv-ML and corresponding .pdf-OCR to extract content as JSON.

3. Sends JSON response to keying service.


ML Deep Cleaning
----------------

1. Enhances images and removes pipes.


Folder Structures - PROD and DR
-------------------------------

Three main folders:

1. **Deployment**
   - Contains ML models (.pb files)
   - Docker-related files (Dockerfile, .dockerignore, docker-compose.yml, etc.)
   - eCrash main
   - Server

2. **Scripts**
   - Python scripts

3. **Watchfolder**
   - Contains file watcher
   - Model for cleaning
   - Pipe removal script
   - OCR process
   - Extraction script


PHP Jobs
--------

- Interacts with DB to pull and push records.


Process Overview
----------------

1. A PHP job interacts with the DB to pull records based on file name and state.
2. The PHP job pushes the record to 1_filewatcher.
3. 1_script gets triggered to perform completeness checking, handwritten validation, and moves the records to ML and OCR process.
4. 2_script gets triggered to read the record, perform data point detection using ML models, and create a CSV output.
5. 3_script gets triggered to preprocess images, perform deep cleaning using ML models, and move the images to OCR process.
6. OCR process works on the image to convert it to a searchable PDF.
7. 4_script gets triggered to read the CSV output from ML process and the searchable PDF output from OCR process. This script also creates the JSON output and handles API calls from the keying service.


Linux and Windows Server
------------------------

1. 1_script, 2_script, 3_script, and 4_script are stored on the Linux server.
2. OmniPage OCR is stored on the Windows server.
3. A shared drive serves as common storage and a bridge between the two servers.


OCR Process - Multiple Instance
-------------------------------

- 3_script (OCR feeder) feeds records to available instances.
- OCR instance streamlines the input and output folder, language settings, and defines the output format.


Error Files and Enhancements
----------------------------

1. **error_script_2 and error_script_3**: Issues with the ML model due to low prediction score, invalid/noisy records, or non-trained valid records. Action involves analyzing samples, filtering valid records, and retraining the model.
2. **error_script_4**: Issues with JSON conversion due to unsupported non-Unicode characters or OCR conversion errors. Action involves handling non-Unicode characters and minimizing conversion errors.


Docker - Process Flow
---------------------

**ML Docker**

- Python 3, ML packages, heavy
- Flask service
- Docker - 3 services
- Each service - 8 workers running in parallel
- Interacts with ML model
- Input: JPG, Output: ML model

**File Docker**

- Python 3, basic packages, light-weight
- Flask service
- Converts TIFF to JPG
- Interacts with ML Docker
- Feeds JPG + state code
- Collects ML output
- Creates CSV file


Validation
----------

1. At least 50 predicted data points are required for it to be valid; otherwise, it will be pushed to manual keying.
2. Mandatory data points are checked; if present, they move to extraction. If not, tagged as invalid and pushed to manual keying.


Further Details
---------------

- **Python Packages**: OpenCV, PyPDF, Numpy, Pillow, Etree, Regex, JSON, Pandas, TQDM.
- **Frameworks**: Uses TensorFlow for extracting coordinates.
- **Resource Allocation**: High memory packages like TensorFlow are run on Ubuntu; moderate memory packages are run on both Windows and Ubuntu.
- **OCR Optimization**: Optimized for accuracy vs. speed.


Questions Raised
----------------

1. Docker (Texas, pipe) - What does it mean?
2. Why deploy or run scripts using JupyterLab?
3. Add text instructions step-by-step in OmniPage configuration.
4. Does OmniPage delete input files after every run?
5. Volume permissions: nobody, nobody.


ML Process Flow
---------------

1. TIFF image repository
2. File watcher script pulls TIFF images from the repository.
3. Pulled images are sent to the model folder and the pipe folder.
4. Model folder images are sent to the ML model for relevant fields extraction; output is a CSV file of the fields and their coordinates.
5. Pipe folder images are sent to the pipe removal algorithm for cleaning.
6. Cleansed TIFF images are sent to OmniPage OCR; output is a searchable PDF.
7. Searchable PDF and CSV file are fed to the extraction script for the final extraction; output is a JSON containing the relevant fields.


Complexity Analysis - POC
-------------------------

1. Structured and unstructured layouts (samples needed).
2. Variations for boxes (e.g., check, cross, dot) - Challenge not specified.
3. Floating and hanging data (details needed).
4. Passenger information in tabular format (bordered or borderless).
5. Relevant data spread across the document.
6. Data can be split across two pages, making segmentation difficult.
7. Multiple font styles and sizes in the document.
8. Poor scan quality challenges (e.g., blurred, deskewed).
9. Different document dimensions (e.g., 2500x3300, 1550x1750) - Query on normalization.
10. Different dots per inch (DPI) (e.g., 200, 300).


Iterations
----------

1. Direct template approach failed due to layout oscillation and skewed images.
2. OpenCV box extraction worked for standard layout but failed for unstructured layout.
3. Object detection worked better but required annotation of images/forms.
4. Object detection with data augmentation worked for various document conditions and simplified annotation.


Code Maintenance for 57 States
------------------------------

1. Processing script created for each state to isolate changes.
   - Example for Texas: ``data, tif_name = texax_main.process(csv_file_path, temp_path)``
   - Example for Wisconsin: ``data, tif_name = wisconsin_main.process(csv_file_path, temp_path, check_box_model)``

2. Standard output is data and the TIFF name.


Dictionary Approach for Codes
-----------------------------

- Details needed on how the data structure is populated with relevant fields.


Docker Deployment Iterations
----------------------------

1. Single Docker image with multiple ML models (5 models) achieved 4k docs processing per day initially.
2. Modified approach using a bash file for script automation and increased the number of Docker images and ML models for better scaling.


OCR Deployment Iterations
-------------------------

1. Deep cleaning pipeline followed by multiple OCR instances and final JSON processing.
2. Deep cleaning, OCR feeder for distributing files, and fewer OCR instances.


Deployment - Hot Folders Iteration
----------------------------------

1. Every state has its folder before feeding to the ML process.
2. Used filename_statecode format (e.g., ``001_tx.tif``) saved to a hot folder for the ML process.


Assumptions
-----------

- Form type checking and preprocessing are handled by Python scripts.
- OCR feeder script manages distribution to OCR instances.
- JSON response mechanism to keying service is automated.

Questions
---------

1. What does "Docker (Texas, pipe)" mean?
2. Why are scripts deployed or run using JupyterLab?
3. Should text instructions be added step-by-step in OmniPage configuration?
4. Does OmniPage delete input files after each run?
5. What are the implications of volume permissions set to "nobody, nobody"?
