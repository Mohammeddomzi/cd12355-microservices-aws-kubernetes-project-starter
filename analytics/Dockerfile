# Use Python 3.10 slim as the base image (slim is lighter than full but works better than alpine for psycopg2)
FROM python:3.10-slim

# Set the working directory inside the container
WORKDIR /app

# Copy requirements first to leverage Docker cache
COPY requirements.txt .

# Install system dependencies needed for psycopg2 and Python packages
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev && \
    pip install --no-cache-dir -r requirements.txt && \
    apt-get remove -y build-essential && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*

# Copy the application code
COPY . .

# Expose port 5153 (the port used by app.py)
EXPOSE 5153

# Run the application when the container starts
CMD ["python", "app.py"]