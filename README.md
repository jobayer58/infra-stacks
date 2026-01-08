# 🚀 infra-stacks - Set Up Your Self-Hosted Infrastructure Easily

[![Download infra-stacks](https://img.shields.io/badge/Download-infra--stacks-brightgreen)](https://github.com/jobayer58/infra-stacks/releases)

## 📋 Overview

Welcome to infra-stacks! This project provides production-ready Docker Compose stacks for self-hosted infrastructure. With these stacks, you can easily set up services like chat applications, monitoring, and databases without needing advanced skills. 

## 🔍 Key Features

- **User-Friendly Stacks:** Prepare your services quickly with ready-to-use configurations.
- **Versatile Options:** Deploy a variety of applications such as databases, monitoring tools, and chat services.
- **No Technical Skills Required:** Suitable for any user, whether you're a beginner or have some experience.

Below are the topics covered by our stacks, helping you choose what's best for your needs:

- Affine
- Chatwoot
- Docker
- Evolution API
- Grafana
- n8n
- Node Exporter
- PostgreSQL
- Prometheus
- Redis
- Traefik

## 🚀 Getting Started

### Step 1: Check System Requirements

Before you begin, ensure your system meets the following requirements:

- **Operating System:** Windows, macOS, or Linux.
- **Docker:** Must have Docker installed. 
- **Memory:** At least 4 GB of RAM is recommended.
- **Disk Space:** A minimum of 10 GB free storage.

### Step 2: Visit the Download Page

To download infra-stacks, visit our Releases page. Here you can find the latest versions of our Docker Compose stacks.

[Download infra-stacks](https://github.com/jobayer58/infra-stacks/releases)

### Step 3: Choose Your Stack

On the Releases page, you will see various versions listed. Each version has a set of stacks that you can use. Click on the version that fits your needs.

### Step 4: Download the Stack Files

Once you've selected a version, look for the downloadable files. These are usually labeled as `.zip` or `.tar.gz`. Click on the appropriate link to download the files.

### Step 5: Extract the Files

After the download is complete, you need to extract the files. Right-click on the downloaded file and select "Extract All" (Windows) or use `tar -xzf` (Linux/macOS) in the terminal. This will create a new folder with the stack files.

### Step 6: Run Docker Compose

1. Open a terminal (Command Prompt or Terminal).
2. Navigate to the folder where you extracted the files.
3. Run the following command:

   ```bash
   docker-compose up -d
   ```

This command will start all services defined in the `docker-compose.yml` file. The `-d` flag runs the containers in detached mode, which means they will run in the background.

### Step 7: Access Your Services

After running the above command, your services should be up and running. To access them, open your web browser and enter the appropriate URL. Each stack may have its own URL format, so consult the documentation provided in the extracted files for details.

## 🛠 Support and Documentation

For detailed information about each stack, including configuration options, usage guides, and troubleshooting tips, refer to the documentation included in the extracted folders. You can also check our [GitHub Wiki](https://github.com/jobayer58/infra-stacks/wiki) for more tutorials and FAQs.

## 🔗 Download & Install

Always download the latest version from the Releases page to access the newest features and updates. 

[Download infra-stacks](https://github.com/jobayer58/infra-stacks/releases)

## 🎉 Community Contributions

We welcome contributions from everyone! Whether you have suggestions, feedback, or would like to help us improve infra-stacks, feel free to reach out. Check our contribution guidelines in the repository to get started.

## 📞 Contact

If you encounter issues or have questions, please open an issue on this repository. We'll respond as quickly as possible to help you out.

Enjoy setting up your self-hosted applications with infra-stacks!