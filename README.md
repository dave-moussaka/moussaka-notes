# moussaka-notes
Notes and useful scripts


	1.	Setup Dependencies
	•	Use requests for HTTP requests.
	•	Use schedule for scheduling tasks.
	•	Use time for retry delays.
	•	Use os and shutil for file handling.
	2.	Service Flow
	1.	Poll the source endpoint using a cron job or a configurable timer.
	2.	Make an HTTP GET request to download a zipped file.
	3.	Save the file to disk.
	4.	Attempt to POST the file to the destination endpoint.
	5.	If the POST request fails, retry after 2 minutes.
	6.	Delete the file after a successful POST.
	3.	Dockerization
	•	Write a Dockerfile to containerize the application.
	•	Deploy the container to OpenShift.

Python Packages and Installation

	1.	Packages
	•	requests for HTTP requests.
	•	schedule for scheduling tasks.
	•	time for sleep functionality.
	2.	Installation
	•	Create a requirements.txt file with the following content:

requests
schedule

	2.	
	•	Install the packages using pip:

pip install -r requirements.txt



Python Script (service.py)

import requests
import schedule
import time
import os
import shutil

# Configuration
SOURCE_URL = 'http://example.com/source.zip'
DESTINATION_URL = 'http://example.com/endpoint'
RETRY_INTERVAL = 120  # Retry every 2 minutes
DOWNLOAD_DIR = '/tmp/downloads'

# Ensure download directory exists
os.makedirs(DOWNLOAD_DIR, exist_ok=True)

def download_file():
    response = requests.get(SOURCE_URL)
    if response.status_code == 200:
        file_path = os.path.join(DOWNLOAD_DIR, 'source.zip')
        with open(file_path, 'wb') as f:
            f.write(response.content)
        return file_path
    else:
        print(f"Failed to download file: {response.status_code}")
        return None

def post_file(file_path):
    with open(file_path, 'rb') as f:
        files = {'file': f}
        response = requests.post(DESTINATION_URL, files=files)
        return response.status_code == 200

def process_file():
    file_path = download_file()
    if file_path:
        success = post_file(file_path)
        if success:
            os.remove(file_path)
            print("File posted successfully and deleted.")
        else:
            print("Failed to post file. Will retry in 2 minutes.")
            schedule.every(RETRY_INTERVAL).seconds.do(retry_post, file_path=file_path)

def retry_post(file_path):
    success = post_file(file_path)
    if success:
        os.remove(file_path)
        print("File posted successfully and deleted.")
        return schedule.CancelJob
    else:
        print("Retrying to post file...")

def main():
    schedule.every(10).minutes.do(process_file)  # Configure as needed

    while True:
        schedule.run_pending()
        time.sleep(1)

if __name__ == "__main__":
    main()

Dockerfile

# Use the official Python image
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy requirements.txt and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the service script
COPY service.py .

# Run the service
CMD ["python", "service.py"]

Running in a Container on OpenShift

	1.	Build the Docker image

docker build -t my-service:latest .


	2.	Push the Docker image to a container registry

docker tag my-service:latest <registry>/<namespace>/my-service:latest
docker push <registry>/<namespace>/my-service:latest


	3.	Deploy the image on OpenShift
	•	Create a deployment configuration:

oc new-app <registry>/<namespace>/my-service:latest --name=my-service


	4.	Expose the service if needed

oc expose svc/my-service



Summary

	•	Packages: requests, schedule
	•	Installation: pip install -r requirements.txt
	•	Script: Handles downloading, saving, posting, and retrying.
	•	Dockerization: Dockerfile to containerize the application.
	•	Deployment: Build, push, and deploy the container to OpenShift.



 1. Python Script (service.py)

This script will hit an API, download a zipped file, and then post it to another endpoint. It will retry if the post fails.

import requests
import os
import time

# Configuration from environment variables
SOURCE_URL = os.getenv('SOURCE_URL', 'http://example.com/source.zip')
DESTINATION_URL = os.getenv('DESTINATION_URL', 'http://example.com/endpoint')
RETRY_INTERVAL = int(os.getenv('RETRY_INTERVAL', '120'))  # Default to 2 minutes
DOWNLOAD_DIR = '/tmp/downloads'

# Ensure download directory exists
os.makedirs(DOWNLOAD_DIR, exist_ok=True)

def download_file():
    response = requests.get(SOURCE_URL)
    if response.status_code == 200:
        file_path = os.path.join(DOWNLOAD_DIR, 'source.zip')
        with open(file_path, 'wb') as f:
            f.write(response.content)
        return file_path
    else:
        print(f"Failed to download file: {response.status_code}")
        return None

def post_file(file_path):
    with open(file_path, 'rb') as f:
        files = {'file': f}
        response = requests.post(DESTINATION_URL, files=files)
        return response.status_code == 200

def process_file():
    file_path = download_file()
    if file_path:
        success = post_file(file_path)
        if success:
            os.remove(file_path)
            print("File posted successfully and deleted.")
        else:
            print("Failed to post file. Will retry in 2 minutes.")
            retry_post(file_path)

def retry_post(file_path):
    while True:
        success = post_file(file_path)
        if success:
            os.remove(file_path)
            print("File posted successfully and deleted.")
            break
        else:
            print("Retrying to post file...")
            time.sleep(RETRY_INTERVAL)

if __name__ == "__main__":
    process_file()

2. Dockerfile

This Dockerfile will package the Python script into a Docker container.

# Use the official Python image
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy requirements.txt and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the service script
COPY service.py .

# Run the service script
CMD ["python", "service.py"]

3. Requirements File (requirements.txt)

Create a requirements.txt file to specify the required Python packages.

requests

4. CronJob YAML (cronjob.yaml)

This YAML file defines the CronJob to run the containerized script at specified intervals.

apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/10 * * * *"  # Runs every 10 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: my-service
            image: <registry>/<namespace>/my-service:latest
            imagePullPolicy: IfNotPresent
            env:
            - name: SOURCE_URL
              value: "http://example.com/source.zip"
            - name: DESTINATION_URL
              value: "http://example.com/endpoint"
            - name: RETRY_INTERVAL
​⬤
