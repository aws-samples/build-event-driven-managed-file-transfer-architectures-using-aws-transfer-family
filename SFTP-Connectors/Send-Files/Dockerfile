# Grab Python3.12 Lambda base image
FROM public.ecr.aws/lambda/python:3.12.2024.01.05.15

# Copy requirements.txt
COPY requirements.txt ${LAMBDA_TASK_ROOT}

# Install the specified packages
RUN pip install -r requirements.txt

# Copy function code
COPY lambda_function.py ${LAMBDA_TASK_ROOT}

# Set the CMD to your handler 
CMD [ "lambda_function.handler" ]