Simple model deployment with kubernetes:
--------------------------------------
Here is the simplest, cleanest possible example of deploying a Hugging Face model on Kubernetes without Triton or complex steps. <br>

We will use: <br>
✅ Model: distilbert-base-uncased <br>
A tiny, fast text classifier model from HuggingFace. <br>

We will run it using:<br>
✅ Framework: FastAPI + Transformers<br>
✅ Container: custom Docker image<br>
✅ Kubernetes Deployment + Service<br>

### Step 1 - install dependencies:
```
# install python3.10-venv package if not installed
apt install python3.10-venv

# create virtual environment
python3 -m venv hf-venv
source hf-venv/bin/activate

# install huggingface-cli - If want to download model manually. (not required in this example. )
pip install --upgrade huggingface_hub
hf download distilbert/distilbert-base-uncased
```

### Step 2 - Create project folder:
```
mkdir hf-k8s-model
cd hf-k8s-model
```

### Step 3 - Create app.py (FastAPI inference server)
```
from fastapi import FastAPI
from pydantic import BaseModel
from transformers import pipeline

app = FastAPI()

# Load HF model
classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased")

class Request(BaseModel):
    text: str

@app.post("/predict")
def predict(req: Request):
    out = classifier(req.text)
    return {"result": out}

```

### Step 4 - Create requirements.txt
```
fastapi
uvicorn
transformers
torch
```

### Step 5 - Create Dockerfile:
Here the model is downloaded from huggingface. thats why we have not downloaded this in step 1.
```
FROM python:3.10-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app
COPY app.py .

# Download the model during build (optional)
RUN python -c "from transformers import pipeline; pipeline('sentiment-analysis', model='distilbert-base-uncased')"

# Expose port
EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 6 - Build and push image on dockerhub.
```
docker build -t <your-dockerhub-user>/hf-demo:latest .
docker push <your-dockerhub-user>/hf-demo:latest
```

### Step 7 - Create kubernetes deployment
Create file : hf-model.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hf-model
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hf-model
  template:
    metadata:
      labels:
        app: hf-model
    spec:
      containers:
      - name: hf-container
        image: <your-dockerhub-user>/hf-demo:latest
        ports:
        - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: hf-model-service
spec:
  type: NodePort
  selector:
    app: hf-model
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: 30080
```
### Deploy:
```
kubectl apply -f hf-model.yaml
kubectl get pods -w
```

### Test model:
```
kubectl get nodes -o wide
```

```
curl -X POST http://<NODE-IP>:30080/predict \
  -H "Content-Type: application/json" \
  -d '{"text":"This is amazing!"}'
```

Your HuggingFace model is now running in Kubernetes! <br>
Simple. No Triton. No GPU requirement.


Auto-scaling using CPU (HPA):
-----------------------------
Step 1 - Create : deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hf-model
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hf-model
  template:
    metadata:
      labels:
        app: hf-model
    spec:
      containers:
      - name: hf-model
        image: yourrepo/hf-sentiment:1
        ports:
        - containerPort: 8000
        resources:
          requests:
            cpu: "200m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: hf-model
spec:
  type: ClusterIP
  selector:
    app: hf-model
  ports:
  - port: 80
    targetPort: 8000
```

Step 2 - Create Horizontal pod autoscaler:
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hf-model-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hf-model
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply changes:
```
kubectl apply -f deployment.yaml
kubectl apply -f hpa.yaml
```

Now test by generating load:
```
while true; do curl -X POST http://10.96.11.14/predict  -H "Content-Type: application/json"  -d '{"text":"hello"}'; done
```
Note: Here i am using cluster IP from host machine, it will only work for the single node cluster. because it is in the same network. <br>

check scaling:
```
kubectl get hpa
```
If output is like this, <br>
<img width="929" height="67" alt="image" src="https://github.com/user-attachments/assets/f7164b12-b9d2-4661-9c31-498afac93223" />

Then there is an issue. <br>
⚠️ HPA cannot read CPU metrics <br>
⚠️ Your cluster does NOT have metrics-server installed <br>
⚠️ So HPA will NEVER autoscale <br>


### Install metrics server:
```
kubectl top pod
```
If it is not showing, <br>

install metrics server:
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Edit:
```
kubectl -n kube-system edit deployment metrics-server
```
Add this under spec.template.spec.containers[0].args:
```
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
```
Then restart,
```
kubectl rollout restart deployment metrics-server -n kube-system
```
Now check,
```
kubectl top pod
```
<img width="656" height="73" alt="image" src="https://github.com/user-attachments/assets/cbc298b6-8d70-4616-8273-b2190f9b49ec" />

```
kubectl get hpa
```
<img width="890" height="68" alt="image" src="https://github.com/user-attachments/assets/ebe4075b-6d27-40dc-a30b-844bfbb0bfe7" />

Now according to this your pods will scale:
<img width="639" height="272" alt="image" src="https://github.com/user-attachments/assets/d5cb1450-7fcb-4274-86db-753e5ec3f58c" />


