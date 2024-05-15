# Define custom function directory
ARG FUNCTION_DIR="/function"

FROM public.ecr.aws/amazonlinux/amazonlinux:2023

# Include global arg in this stage of the build
ARG FUNCTION_DIR

# Install GNUPG and other dependencies
RUN dnf update -y \
    && dnf install -y python3 python3-pip \
    && dnf install --allowerasing -y gnupg2-full

# Copy function code
RUN mkdir -p ${FUNCTION_DIR}
COPY . ${FUNCTION_DIR}

# Install the function's dependencies
RUN pip3 install \
    --target ${FUNCTION_DIR} \
        awslambdaric \
        boto3 \
        python-gnupg \
        botocore

# Set working directory to function root directory
WORKDIR ${FUNCTION_DIR}

# Set runtime interface client as default command for the container runtime
ENTRYPOINT [ "/usr/bin/python3", "-m", "awslambdaric" ]

# Pass the name of the function handler as an argument to the runtime
CMD [ "lambda_function.handler" ]