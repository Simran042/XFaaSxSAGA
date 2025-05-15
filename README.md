This file outlines the procedure for deploying **XFaaS** and **XFBench** in a single-cloud and multi-cloud environment. The aim is to set up a functioning serverless execution framework that facilitates benchmark experiments on public cloud providers like AWS and Azure.

---

## Single Cloud Deployment Setup

### Running XFBench

To begin with, clone the XFaaS and XFBench repositories, and run XFBench inside a Docker container.

```bash
# Clone the XFaaS repository
git clone -b CCGRID2024 https://github.com/dream-lab/XFaaS.git

# Navigate into XFaaS
cd XFaaS

# Authenticate with Azure
az login -u <username> -p <password>

# Set Azure subscription
export AZURE_SUBSCRIPTION_ID=<your-subscription-id>

# Configure AWS CLI
aws configure

# Clone XFBench
git clone -b CCGRID2024 https://github.com/dream-lab/XFBench.git

# Copy XFaaS serwo directory
cp -r XFaaS/serwo /XFBench/bin/

# Export environment variables
export XFBENCH_DIR=/XFBench
export XFAAS_DIR=/XFaaS
export XFAAS_WF_DIR=/XFBench/workflows/custom_workflows/graph_processing_wf

# Move to XFBench
cd /XFBench

# Install Python dependencies
pip3 install -r requirements.txt
```

---

### Setting up Dockerfile Environment on Host Machine

Install the following prerequisites on your host machine:

```bash
# Update system and install essentials
sudo apt update
sudo apt install -y python3.9 python3-pip git curl unzip wget gnupg software-properties-common

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -q awscliv2.zip
sudo ./aws/install
rm -rf awscliv2.zip aws

# Install AWS SAM CLI
curl -LO https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
unzip -q aws-sam-cli-linux-x86_64.zip -d sam-installation
sudo ./sam-installation/install
rm -rf aws-sam-cli-linux-x86_64.zip sam-installation

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install Azure Functions Core Tools
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo mv microsoft.gpg /etc/apt/trusted.gpg.d/microsoft.gpg

source /etc/os-release
echo "deb [arch=amd64] https://packages.microsoft.com/repos/microsoft-ubuntu-${VERSION_CODENAME}-prod ${VERSION_CODENAME} main" | sudo tee /etc/apt/sources.list.d/dotnetdev.list

sudo apt update
sudo apt install -y azure-functions-core-tools-4

# Install Java and Apache JMeter
sudo apt install -y openjdk-11-jdk
curl -LO https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.6.2.tgz
tar xf apache-jmeter-5.6.2.tgz
rm apache-jmeter-5.6.2.tgz

# Add JMeter to PATH
echo 'export PATH="$PATH:$HOME/apache-jmeter-5.6.2/bin"' >> ~/.bashrc
source ~/.bashrc

# Verify installations
aws --version
sam --version
az --version
func --version
java -version
jmeter -v
```

---

### Running XFaaS

#### Prerequisites

* A publicly accessible client machine (e.g., AWS EC2)
* Apache JMeter 5.6.2 installed
* Config file: `serwo/config/client_config.json`

#### Example config file

```json
{
  "<client_key>": {
    "server_ip": "xx.xx.xx.xx",
    "server_user_id": "ubuntu",
    "server_pem_file_path": "/path/to/key.pem"
  }
}
```

#### Environment Setup

```bash
export AZURE_SUBSCRIPTION_ID=<your-subscription-id>
export XFAAS_WF_DIR=$XFAAS_DIR/workloads

pip3 install -r requirements.txt
```

#### Run Example Experiment

```bash
python3 bin/serwo/xfbench_run.py \
  --csp aws \
  --region us-east-1 \
  --max-rps 1 \
  --duration 10 \
  --payload-size small \
  --dynamism static \
  --wf-name graph \
  --wf-user-directory /XFBench/workflows/custom_workflows/graph_processing_wf \
  --dag-file-name dag.json \
  --teardown-flag 0 \
  --client-key localhost
```

```bash
python3 serwo/xfaas_run_benchmark.py \
  --csp aws \
  --region us-east-1 \
  --dynamism alibaba \
  --max-rps 17 \
  --duration 300 \
  --payload-size medium \
  --wf-user-directory /home/simran/Desktop/XFBench/XFBench/workflows/custom_workflows/graph_processing_wf \
  --path-to-client-config /home/simran/Desktop/XFBench/XFaaS/serwo/config/client_config.json \
  --dag-file-name dag.json \
  --teardown-flag 0 \
  --client-key ~/.ssh/BTP_Team29_keypair.pem
```

#### Notes

* `--teardown-flag 0` retains cloud resources after the experiment
* More examples can be found in:

  * `XFBench/scripts/`
  * `XFBench/bin/serwo/exp_runner.py`

---

## Multi-Cloud Deployment Setup

### Prerequisites

Ensure all prerequisites for single-cloud setup are completed before proceeding with multi-cloud deployment.

---

### Setting Up SAGA

```bash
# Navigate to SAGA directory
cd saga

# Install SAGA in editable mode
pip install -e ./src
```

#### Common Error and Fix

If you encounter this error:

```text
ImportError: Failed to import pygraphviz: "module 'pygraphviz' has no attribute '__path__'"
```

It likely means `pygraphviz` is not properly installed. To fix:

```bash
sudo apt-get install graphviz graphviz-dev

pip uninstall pygraphviz -y
pip install pygraphviz --no-binary pygraphviz \
  --install-option="--include-path=/usr/include/graphviz" \
  --install-option="--library-path=/usr/lib/graphviz"
```

Verify installation:

```python
import pygraphviz
print(pygraphviz.__version__)
```

---

### Running XFaaS for Multi-Cloud Workflows

To deploy and evaluate workflows across multiple clouds using different scheduling strategies, use the following structure:

```bash
python serwo_create_multicloud_statemachine.py \
  <workflow-path> \
  <dag-file-name> \
  <number-of-parts> \
  [<Scheduler1> <Scheduler2> ...]
```

#### Example:

```bash
python serwo_create_multicloud_statemachine.py \
  /XFaaS/serwo/examples/smart-grid-multicloud \
  dag-description.json \
  2 \
  heft
```

#### Notes

* Replace placeholders like `<workflow-path>` with actual paths.
* The script supports:

  * **Single Scheduler:** Use one scheduler (e.g., `heft`)
  * **Multiple Schedulers:** Provide multiple names (e.g., `heft minmin`)
  * **All Schedulers:** Use `all` to run all available ones
