# moussaka-notes
Notes and useful scripts

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
