# YOLOv8 Object Detection via AWS Cloud Pipeline

This repository presents a cloud-based deployment pipeline for **YOLOv8 object detection** on **AWS ECS/Fargate**, designed to evaluate how scalable cloud infrastructure performs relative to an EC2-style baseline under simulated live request traffic.

The project focuses not only on model inference, but also on **containerized deployment**, **load-balanced serving**, **cloud monitoring**, and **performance evaluation** for real-time computer vision workloads.

> **Note:** This repository is a fork of a team project. I am keeping it in my GitHub because it reflects my direct contribution to the inference and deployment side of the pipeline. This README highlights my role and the overall project context.

## Project Overview

Modern object detection systems often need more than a strong model—they need to handle changing traffic patterns, maintain stable latency, and scale efficiently. This project uses **YOLOv8** for object detection and deploys it in a cloud-native AWS environment to study:

- inference latency
- throughput under varying request patterns
- resource utilization
- behavior under load balancing and autoscaling
- tradeoffs between scalable deployment and a simpler baseline setup

The workload is based on the **COCO 2017 validation dataset**, submitted through a simulated live request stream.

## My Contribution

In this collaborative project, my primary contribution was on the **model serving and deployment preparation** side. I was responsible for:

- setting up **YOLOv8 inference**
- wrapping the model in a **lightweight Flask API**
- **containerizing** the service with Docker
- preparing the image for deployment through **Amazon ECR**

## Project Goals

The pipeline was designed to compare a cloud-native deployment against a baseline implementation while measuring:

- **latency**
- **throughput**
- **resource utilization**
- **tail latency under bursty load**
- **cost-performance tradeoffs**

The larger goal was to understand whether autoscaling and load balancing improve inference behavior for object detection workloads under changing demand.

## Run YOLOv8 using AWS ECS + Fargate Pipeline

This project runs the object detection model YOLOv8 on an AWS ECS Fargate Pipeline levaraging the auto scaling and load balancing capabilities of AWS and comparing it to an AWS EC2 Baseline. The model is run against the COCO 2017 Validation data using a live simulation.

**Files:**
- `run_fargate.sh`: Will create and configure all required resources for AWS CLI. All resources created in the file are deleted at the end.
- `run_fargate_partial.sh`: Will create and configure all required resources for AWS CLI. Only core architecture resources created in the file are deleted at the end.
- `yolo-api_amd64-20251104`: Container image to be uploaded to AWS ECR.
- `dashboard.json`: CloudWatch Dashboard configuration with metrics and alarms. ARNs will be updated appropriately when run_fargate.sh is run.
- `requirements.txt`: Requirements to install for running the live simulation.
- `live_request_sim.py`: Script to submit image requests iteratively in varying loads and frequencies.
- `coco_val_manifest.json`: Contains all requests for the 5,000 images in the COCO 2017 Validation dataset.
- `request_log.ndjson`: Contains the responses after requests are sent, including the class detected in the image (person, dog, plant, etc).


The file `run_fargate.sh` uses AWC CLI to automatically set up all the required resources that allow you to run the image yolo-api_amd64-20251104.tar from your ECR repo (yolo-api), submit the requests, and clean up all the created resources. You can instead `run_fargate_partial.sh` to set up all required resources and only clean up some resources while maintaining the core architecture for a future re-run (retains IAM role, ECS cluster, VPC, subnets, route table, internet gateway, security group, and CloudWatch Alarms).

Note that Steps 1-2 (AWS CLI setup and loading image to ECR) only have to be completed once, after which you can begin running the pipeline at Step 3.

This repository uses **Git Large File Storage (LFS)** to store yolo-api_amd64-20251104.tar. To ensure it downloads correctly, please install Git LFS before cloning. 

If Git LFS is not installed, the file may appear as a small pointer file after cloning. In this case, download the tar file separately (≈ 558 MB) using this link (https://drive.google.com/drive/folders/16KduWvdqoBbY8HDFhBqgTvzbuIKCZRF-?usp=sharing) and replace the pointer file in the repository directory before continuing.

- `macOS`: brew install git-lfs
- `Windows`: [https://git-lfs.com](https://git-lfs.com)

Then run:
```bash
   git lfs install
```

## Step 1 - Set up AWS CLI credentials

If you haven’t used the AWS CLI on this machine yet, do this first:

1. Create an access key for your IAM user (your AWS account):
   - AWS Console → IAM → Users → *your user* → Security credentials → Create access key
   - Choose Command Line Interface (CLI), then copy both:
     - Access key ID
     - Secret access key

2. Configure the AWS CLI in Terminal:
   ```bash
   aws configure
   ```

   Enter:
   ```
   AWS Access Key ID [None]: <YOUR_ACCESS_KEY_ID>
   AWS Secret Access Key [None]: <YOUR_SECRET_ACCESS_KEY>
   Default region name [None]: us-east-1
   Default output format [None]: json
   ```
### Important Region Requirement:
Make sure the AWS Console region is set to N. Virginia (us-east-1) when logging in with the IAM user. All resources (ECS, ALB, CloudWatch dashboard, and alarms) are created in us-east-1, and they will not be visible in other regions.

3. Verify your CLI is connected to your account:
   ```bash
   aws sts get-caller-identity
   ```
   You should see JSON with your Account number and ARN.

Once this is done, you can log in to ECR and push images.

## Step 2 - Load the Image and push to ECR

### Load the Docker image

1. Make sure Docker Desktop is installed and running. 

2. Load the image into Docker:
   ```bash
   docker load -i yolo-api_amd64-20251104.tar
   ```
   You’ll see something like:
   ```
   Loaded image: yolo-api:amd64-20251104
   ```

3. Verify architecture
   ```bash
   docker inspect yolo-api:amd64-20251104 --format '{{.Architecture}}'
   ```
   You should see 'amd64'
   
The image is now available in your local Docker system.

---

### Retag it for your AWS account

1. Replace `<YOUR_ACCOUNT_ID>` with your own AWS account ID.

2. Run the following commands:
   ```bash
   aws ecr create-repository --repository-name "yolo-api" --region "us-east-1" >/dev/null 2>&1 || true

   aws ecr get-login-password --region "us-east-1" \
   | docker login --username AWS --password-stdin "<YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com"

   docker tag "yolo-api:amd64-20251104" "<YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/yolo-api:amd64-20251104"
   docker push "<YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/yolo-api:amd64-20251104"
   ```

3. You can confirm using the below code.
   ```bash
   docker images | grep yolo-api
   ```
You’ll see something like -

    ```
    REPOSITORY        TAG           IMAGE ID       SIZE

    yolo-api     amd64-20251104     abc123...      558MB
    ```

## Step 3 - Run the AWS pipeline

1. Create a small `.env` file with these lines:
   ```
   MODEL_NAME=yolov8n
   CONF_THRESHOLD=0.25
   DOWNLOAD_TIMEOUT_S=8
   MAX_IMAGE_MB=10
   PORT=8080
   ```

2. Run the script:
```bash
   sh run_fargate.sh
```

If everything is set up successfully, you will see something like:

```
🔹 Using account: <YOUR_ACCOUNT_ID> in region: us-east-1
🔹 Image: <YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/yolo-api:amd64-20251104
🔹 Port: 8080
Using existing ECS cluster: temp-fargate-cluster
✅ Using existing IAM role: ecsTaskExecutionRole
✅ Using default VPC: vpc-...
✅ Created Internet Gateway: igw-...
✅ Using existing Route Table: rtb-...
✅ Created first subnet: subnet-... (us-east-1a) with CIDR .../24
✅ Created second subnet: subnet-... (us-east-1b) with CIDR .../24
✅ Created security group: sg-...
✅ Target group created: arn:aws:elasticloadbalancing:us-east-1:...
✅ ALB created: arn:aws:elasticloadbalancing:us-east-1:...
✅ ALB Listener created: arn:aws:elasticloadbalancing:us-east-1:...
✅ Task definition already exists: ...
✅ ECS service created: ...
⏳ Waiting 5 minutes for ECS tasks to become healthy in the Target Group...
🔄 Still waiting... (attempt 1)
🔄 Still waiting... (attempt 2)
🔄 Still waiting... (attempt 3)
🔄 Still waiting... (attempt 4)
✅ ECS task(s) are healthy!
📈 Configuring auto scaling for ECS service...

✅ Service is running!
🌍 Public IP: temp-fargate-alb-27906923.us-east-1.elb.amazonaws.com:8080
💡 Test it with:
   curl -X POST http://temp-fargate-alb-27906923.us-east-1.elb.amazonaws.com:8080/predict_json -H 'Content-Type: application/json' -d '{"key":"value"}'

Press ENTER when done testing to clean everything up... 
```

The **PUBLIC_IP** may be different with each execution of the script.

In a separate terminal, submit your curl command using the returned **PUBLIC_IP**. Your command for a single image will look like

```bash
curl -s -X POST http://temp-fargate-alb-27906923.us-east-1.elb.amazonaws.com:8080/predict_json \
  -H "Content-Type: application/json" \
  -d '{
        "req_id": "test-1",
        "image_id": 397133,
        "coco_url": "http://images.cocodataset.org/val2017/000000397133.jpg"
      }'
```

You’ll get a JSON response with the detection and timing for the specified image_id.

## Step 4 - Submitting multiple requests simulataneously

1. From the root directory of the repository, install Python dependencies:

 ```bash
   pip install -r requirements.txt
```
2. Then navigate to the live request directory:
   
 ```bash
  cd live_request
```

4. Create a `.env` file with the following variable

```
API_BASE_URL=http://temp-fargate-alb-27906923.us-east-1.elb.amazonaws.com:8080
```

This is the IP and port returned when you set up the AWS envrionment in Step 3.

### Run the request simulator 

From a separate terminal, you can test the application by sending loads with different latencies and volume (number of image requests up to 5,000). You can also run from multiple terminals in parallel to simulate multiple users.

**Modes**:
- *Quiet*: Randomd elay between requests within 2.0 - 6.0 seconds.
- *Sustained*: Random delay between requests within 0.2 - 0.5 seconds.
- *Burst*: Random delay between requests within 0.02 - 0.12 seconds.

**Example commands:**

```
python3 live_request_sim.py --mode quiet --limit 50 --manifest "coco_val_manifest.json"

# sustained mode
python3 live_request_sim.py --mode sustained --limit 100 --manifest "coco_val_manifest.json"

# burst mode
python3 live_request_sim.py --mode burst --limit 500 --manifest "coco_val_manifest.json"
```

**Output:**
- `request_log.ndjson` will be written containing one JSON object per line with `status`, `latency_ms`, `req_id`, `detections`. etc.
- The simulator logs summary p50/p95/p99 and mean latencies to stdout (or logging) as well.

### CloudWatch Dashboard

When you run the ECS Fargate pipeline, the dashboard is automatically created via the shell script. The ARNs for Target Group and ALB will be automatically udpated with each re-run. Here you can track all the metrics tied to the auto scaling policies and alarms.


## Step 5 - Clean up

Return to the original terminal from where you ran `run_fargate.sh` and hit 'Enter'. This will begin the clean up process and delete all the resources that were created in that script. You will see something as follow:

```
🧹 Cleaning up resources...
✅ ECS service deleted
✅ Tasks stopped
✅ ECS cluster deleted
✅ Listener deleted
✅ Target Group deleted
✅ Load Balancer deleted
✅ IAM policy detached
✅ IAM role deleted
⏳ Waiting for network interfaces to detach from subnet-...
✅ Subnet 1 deleted
⏳ Waiting for network interfaces to detach from subnet-...
⏳ Waiting for network interfaces to detach from subnet-...
✅ Subnet 2 deleted
✅ Internet Gateway deleted
✅ Route Table deleted
✅ Security group deleted
✅ Temporary files cleaned up
✅ Cleanup complete. All resources deleted.
```

You will have to manually delete your ECR repo.

Verify everything is deleted.



## Run YOLOv8 using EC2 Baseline

This section explains the **EC2 baseline** used to compare against the AWS pipeline.

Run a **Flask API** directly on one EC2 instance (using Python command in the terminal). On the local machine, the file `client_upload.py` was used to send different traffic patterns (Quiet, Steady, Increasing, High) to this `/predict` endpoint and log the results to CSV files.

The goal is to see how a single EC2 machine behaves under different loads.

---

### Files used (for this baseline)

- `client_upload.py` – client script that sends requests and logs responses into CSV files.
- `coco_val_manifest.json` – COCO 2017 validation manifest used as the workload.
- `requirements.txt`

---

> Note: The API is started directly with a small inline Flask script on the EC2 instance.

---

### Step 1 – Launch the EC2 instance

1. In the AWS Console, go to **EC2 → Instances → Launch instances**.
2. Suggested settings:

   * **Name**: `ec2-baseline`
   * **AMI**: Amazon Linux 2 (64-bit x86)
   * **Instance type**: any general-purpose instance with at least 2 vCPUs and 4–8 GiB RAM (for example, `t3.large`).
3. **Key pair**: create or choose a key pair so you can SSH into the instance.
4. **Security group**:

   * Allow **SSH (port 22)** from your IP (to log in).
   * Allow **TCP port 5000** from your IP (this is where the Flask `/predict` endpoint will listen).
5. Launch the instance and wait until it is in the **Running** state.
6. Note the **Public IPv4 address** – we’ll refer to this as `<EC2_PUBLIC_IP>`.

---

### Step 2 – Connect to EC2 and install dependencies

From your local machine:

```bash
ssh -i /path/to/your-key.pem ec2-user@<EC2_PUBLIC_IP>
```

On the EC2 instance, install basic tools:

```bash
sudo yum update -y
sudo yum install -y git python3-pip
```

(If you are using `requirements.txt`, you can later do `pip3 install -r requirements.txt`.
For the minimal baseline, we mainly need `flask` and `requests` on the EC2 side.)

Install the Python packages needed for the API:

```bash
pip3 install flask requests
```

---

### Step 3 – Make sure CloudWatch agent is running

For the experiments, the EC2 instance should be sending metrics to CloudWatch.

If the **Amazon CloudWatch Agent** has already been installed and configured earlier, you can confirm it is running with:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

You should see it in an `active (running)` state.
(If it is not running, start it with `sudo systemctl start amazon-cloudwatch-agent` after it has been configured.)

---

### Step 4 – Start the inline Flask `/predict` API on EC2

On the EC2 instance, in your terminal, start a very small Flask app inline using:

```bash
python3 - <<'PY'
from flask import Flask, request, jsonify
app = Flask(__name__)

@app.post("/predict")
def predict():
    data = request.get_json(force=True, silent=True) or {}
    # Baseline behavior: echo back whatever JSON we receive
    return jsonify(ok=True, got=data), 200

app.run(host="0.0.0.0", port=5000)
PY
```

Notes:

* This command:

  * Creates a Flask app.
  * Defines a `POST /predict` endpoint.
  * Reads the JSON body and echoes it back under `got`.
  * Listens on `0.0.0.0:5000` so we can reach it from outside the instance.
* This is a simple baseline: it does not run a model. It is just used as a controlled, lightweight endpoint for load testing with `client_upload.py`.

Keep this terminal window open while running experiments (the server will stop if you close it or hit `Ctrl + C`).

You can quickly test the endpoint from EC2 itself:

```bash
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"req_id": "test-ec2", "hello": "world"}'
```

You should get a response like:

```json
{
  "ok": true,
  "got": {
    "req_id": "test-ec2",
    "hello": "world"
  }
}
```

---

### Step 5 – Prepare the client on your local machine

On your local machine (or any machine that can reach `<EC2_PUBLIC_IP>` on port 5000):

1. Clone this repository and move into it:

   ```bash
   git clone <THIS_REPO_URL>
   cd <THIS_REPO_FOLDER>
   ```

2. Make sure `client_upload.py` and `coco_val_manifest.json` are in this folder.

3. Install the client dependencies (for example):

   ```bash
   pip install requests pandas python-dotenv
   ```

4. Export the API URL environment variable so the client knows where to send requests:

   ```bash
   export API=http://<EC2_PUBLIC_IP>:5000/predict
   ```

5. (Optional) Confirm the script works:

   ```bash
   python client_upload.py --help
   ```

---

### Step 6 – Run the EC2 baseline traffic patterns

All commands below are run **from your local machine**, in the repo folder, with:

* `API` set to `http://<EC2_PUBLIC_IP>:5000/predict`
* `coco_val_manifest.json` present
* The Flask server running on EC2 as described above

Each run:

* Reads image metadata from `coco_val_manifest.json`.
* Sends HTTP POST requests to `$API`.
* Logs responses (including latency) to a CSV file.
* Uses `--tag` to label the scenario.

#### 6.1 Quiet load

Low, sparse traffic to simulate a quiet period:

```bash
python client_upload.py \
  --mode quiet --api "$API" \
  --manifest coco_val_manifest.json \
  --limit 300 --rps 0.25 \
  --csv quiet_ec2.csv --tag quiet-ec2
```

* `limit = 300` → total number of requests.
* `rps = 0.25` → around 1 request every 4 seconds.

---

#### 6.2 Steady load

Constant moderate traffic. Run it twice to get two comparable samples.

```bash
# Steady run 1
python client_upload.py \
  --mode sustained --api "$API" \
  --manifest coco_val_manifest.json \
  --limit 3000 --rps 3 \
  --csv steady_ec2.csv --tag steady-ec2

# Steady run 2
python client_upload.py \
  --mode sustained --api "$API" \
  --manifest coco_val_manifest.json \
  --limit 3000 --rps 3 \
  --csv steady_ec3.csv --tag steady-ec3
```

---

#### 6.3 Increasing load

Start with sustained load, then add bursts separated by short pauses.

```bash
# Sustained phase 1
python client_upload.py --mode sustained --api "$API" \
  --manifest coco_val_manifest.json --limit 1500 --rps 3 \
  --csv inc1_ec2.csv --tag inc1-ec2

# Sustained phase 2
python client_upload.py --mode sustained --api "$API" \
  --manifest coco_val_manifest.json --limit 1500 --rps 3 \
  --csv inc2_ec2.csv --tag inc2-ec2

# Burst phases
sleep 120
python client_upload.py --mode burst --api "$API" \
  --manifest coco_val_manifest.json --limit 500 --rps 14 --burst-size 500 \
  --csv incb1_ec2.csv --tag incb1-ec2

sleep 60
python client_upload.py --mode burst --api "$API" \
  --manifest coco_val_manifest.json --limit 1000 --rps 14 --burst-size 1000 \
  --csv incb2_ec2.csv --tag incb2-ec2

sleep 30
python client_upload.py --mode burst --api "$API" \
  --manifest coco_val_manifest.json --limit 500 --rps 14 --burst-size 500 \
  --csv incb3_ec2.csv --tag incb3-ec2
```

---

#### 6.4 High load

Keeps the instance busy for longer with a mix of sustained and bursty traffic:

```bash
# High sustained phases
python client_upload.py --mode sustained --api "$API" \
  --manifest coco_val_manifest.json --limit 1500 --rps 3 \
  --csv high1_ec2.csv --tag high1-ec2

python client_upload.py --mode sustained --api "$API" \
  --manifest coco_val_manifest.json --limit 1500 --rps 3 \
  --csv high2_ec2.csv --tag high2-ec2

# High burst phases
sleep 120
python client_upload.py --mode burst --api "$API" \
  --manifest coco_val_manifest.json --limit 1000 --rps 14 --burst-size 1000 \
  --csv highb1_ec2.csv --tag highb1-ec2

sleep 120
python client_upload.py --mode burst --api "$API" \
  --manifest coco_val_manifest.json --limit 1000 --rps 14 --burst-size 1000 \
  --csv highb2_ec2.csv --tag highb2-ec2

sleep 45
python client_upload.py --mode burst --api "$API" \
  --manifest coco_val_manifest.json --limit 500 --rps 14 --burst-size 500 \
  --csv highb3_ec2.csv --tag highb3-ec2

sleep 60
python client_upload.py --mode burst --api "$API" \
  --manifest coco_val_manifest.json --limit 500 --rps 14 --burst-size 500 \
  --csv highb4_ec2.csv --tag highb4-ec2

sleep 165
python client_upload.py --mode burst --api "$API" \
  --manifest coco_val_manifest.json --limit 1000 --rps 14 --burst-size 500 \
  --csv highb5_ec2.csv --tag highb5-ec2
```

All of these runs generate the `*_ec2.csv` files that are use to compute metrics like p90/p95/p99 latency and error rates for the EC2 baseline, which are also shown on the terminal.

---

### Step 7 – Stop the baseline

When you are done:

1. On EC2, stop the Flask server by pressing `Ctrl + C` in the terminal where the inline `python3 - <<'PY'` command is running.
2. In the AWS Console, **stop or terminate** the EC2 instance so it does not keep running and generating charges.

## What This Repository Demonstrates

This project highlights experience with:

- ML model serving
- API-based inference workflows
- Dockerized deployment
- AWS-based cloud infrastructure
- scalable computer vision systems
- evaluation of deployment behavior under simulated live load

## Future Improvements

Potential next steps include:

- adding a more detailed architecture diagram
- documenting benchmark results directly in the README
- including screenshots of CloudWatch dashboards
- comparing ECS/Fargate with other deployment options such as EC2-only or Lambda-based inference
- adding CI/CD automation for deployment

## Acknowledgment

This was a collaborative course project. I maintain this fork to document my role in the inference-serving and containerization workflow while preserving the original team repository structure.
