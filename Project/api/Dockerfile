# Use the official Python image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Copy files
COPY requirements.txt ./
RUN pip install -r requirements.txt
COPY . .

# Expose port and run app
ENV PORT=8080
CMD ["python", "main.py"]
