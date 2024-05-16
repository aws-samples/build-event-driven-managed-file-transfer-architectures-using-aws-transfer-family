# Automated File Processing for Transfer Family

## Welcome!

This repository will be used to distribute sample event driven architectures for Managed File Transfer (MFT) using AWS Transfer Family.

We plan to add new folders to this repository, which will each contain different architectural sample based on unique use cases relating to MFT workloads. These folders will contain CloudFormation templates you will be able to download and use to deploy the documented sample architectures in your own environment.

These samples are intended to help you more fully understand how to use AWS Transfer Family along with Amazon EventBridge, AWS Step Functions, AWS Lambda, and other services to build powerful event driven architectures to enable MFT with your business partners and users. Each folder will have its own corresponding README file detailing how to deploy the solution.

For more information on how this repository is setup, please refer below.

## Repository Folder Structure

    .
    ├── SFTP-Connectors             # Architectures using SFTP Connectors
    │   └── Send-Files              # Processing, encrypting, and sending files using SFTP Connectors
    │   └── Retrieve-Files          # Retrieving, decrypting, and processing files using SFTP Connectors (Work in Progress)

## Solutions

For processing, encrypting, and <strong>sending</strong> files using AWS Transfer Family SFTP connectors, refer here: [Sending Files using SFTP Connectors](https://aws.amazon.com/blogs/storage/architecting-secure-and-compliant-managed-file-transfers-with-aws-transfer-family-sftp-connectors-and-pgp-encryption/)

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
