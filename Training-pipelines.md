# Training Pipelines - Azure ML

> **Exam Weight:** Part of "Train and deploy models (25–30%)"  
> **Last Updated:** 2025-06-15  
> **Status:** #studying #azure-ml #pipelines

## 📋 Overview

Training pipelines in Azure ML allow you to automate and orchestrate machine learning workflows. They help you create reproducible, scalable ML processes.

### Key Concepts
- **Pipeline**: A workflow of ML steps that can be executed as a single unit
- **Component**: Reusable piece of code that performs a specific task
- **Step**: Individual operation in a pipeline
- **Datastore**: Storage location for data used in pipelines

---

## 🏗️ Pipeline Components

### Types of Components

#### 1. Built-in Components
```yaml
# Example: Data preprocessing component
- name: preprocess_data
  type: command
  inputs:
    raw_data: 
      type: uri_folder
  outputs:
    processed_data:
      type: uri_folder
```

#### 2. Custom Components
- Written in Python, R, or other languages
- Packaged as Docker containers
- Reusable across different pipelines

```python
# Example custom component
from azure.ai.ml import command
from azure.ai.ml.entities import Environment

@command(
    inputs=dict(
        data=Input(type="uri_folder"),
        test_split_ratio=Input(type="number", default=0.2),
    ),
    outputs=dict(
        train_data=Output(type="uri_folder"),
        test_data=Output(type="uri_folder"),
    ),
    environment="my-sklearn-env",
)
def train_test_split_component(data, test_split_ratio, train_data, test_data):
    # Component logic here
    pass
```

---

## 🔧 Creating Pipelines

### Method 1: Using Azure ML Studio (GUI)
1. Open Azure ML Studio
2. Go to **Designer** tab
3. Drag and drop components
4. Connect components with data flows
5. Configure component parameters
6. Submit pipeline run

### Method 2: Using Python SDK v2

```python
from azure.ai.ml import MLClient, Input, Output
from azure.ai.ml.dsl import pipeline
from azure.ai.ml.entities import Environment

# Initialize ML Client
ml_client = MLClient(
    credential=DefaultAzureCredential(),
    subscription_id="your-subscription-id",
    resource_group_name="your-resource-group",
    workspace_name="your-workspace-name"
)

@pipeline(
    description="Training pipeline for classification model",
    tags={"team": "data-science", "project": "customer-churn"},
)
def training_pipeline(pipeline_input_data, test_split_ratio: float = 0.2):
    
    # Step 1: Data preprocessing
    preprocess_step = preprocess_data_component(
        raw_data=pipeline_input_data
    )
    
    # Step 2: Split data
    split_step = train_test_split_component(
        data=preprocess_step.outputs.processed_data,
        test_split_ratio=test_split_ratio
    )
    
    # Step 3: Train model
    train_step = train_model_component(
        train_data=split_step.outputs.train_data
    )
    
    # Step 4: Evaluate model
    evaluate_step = evaluate_model_component(
        model=train_step.outputs.model,
        test_data=split_step.outputs.test_data
    )
    
    # Return pipeline outputs
    return {
        "trained_model": train_step.outputs.model,
        "evaluation_results": evaluate_step.outputs.evaluation_results
    }

# Create pipeline instance
pipeline_job = training_pipeline(
    pipeline_input_data=Input(
        type="uri_folder", 
        path="azureml://datastores/workspaceblobstore/paths/data/"
    ),
    test_split_ratio=0.3
)

# Submit pipeline
pipeline_run = ml_client.jobs.create_or_update(
    pipeline_job, 
    experiment_name="training_pipeline_experiment"
)
```

### Method 3: Using YAML Definition

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/pipelineJob.schema.json
type: pipeline
display_name: "Training Pipeline"
description: "Pipeline for model training and evaluation"

inputs:
  input_data:
    type: uri_folder
    path: azureml://datastores/workspaceblobstore/paths/data/

outputs:
  trained_model:
    type: uri_folder

jobs:
  prep_data:
    type: command
    component: azureml:data_prep:1
    inputs:
      raw_data: ${{parent.inputs.input_data}}
    outputs:
      processed_data: 
        type: uri_folder

  train_model:
    type: command
    component: azureml:train_sklearn:1
    inputs:
      training_data: ${{parent.jobs.prep_data.outputs.processed_data}}
    outputs:
      model:
        type: uri_folder

  evaluate_model:
    type: command
    component: azureml:evaluate_model:1
    inputs:
      model: ${{parent.jobs.train_model.outputs.model}}
      test_data: ${{parent.jobs.prep_data.outputs.processed_data}}
```

---

## 📊 Passing Data Between Steps

### 1. Using Outputs and Inputs
```python
# Producer step
step1 = component1(input_data=pipeline_input)

# Consumer step - uses output from step1
step2 = component2(processed_data=step1.outputs.processed_data)
```

### 2. Data Types
- **uri_folder**: Folder containing multiple files
- **uri_file**: Single file
- **mltable**: Structured data table
- **mlflow_model**: MLflow model format

### 3. Data Locations
```python
# From datastore
Input(type="uri_folder", path="azureml://datastores/mydatastore/paths/data/")

# From previous step
Input(type="uri_folder", path=previous_step.outputs.output_data)

# Local file
Input(type="uri_file", path="./local_file.csv")
```

---

## ⚙️ Running and Scheduling Pipelines

### Manual Execution
```python
# Submit pipeline job
pipeline_run = ml_client.jobs.create_or_update(
    pipeline_job,
    experiment_name="my_experiment"
)

# Monitor run status
print(f"Pipeline run status: {pipeline_run.status}")
print(f"Run URL: {pipeline_run.studio_url}")
```

### Scheduled Execution
```python
from azure.ai.ml.entities import JobSchedule, RecurrencePattern, RecurrenceTrigger

# Create schedule
schedule = JobSchedule(
    name="weekly_training_schedule",
    trigger=RecurrenceTrigger(
        frequency="week",
        interval=1,
        schedule=RecurrencePattern(hours=[2], minutes=[0])  # 2:00 AM
    ),
    create_job=pipeline_job
)

# Submit schedule
ml_client.schedules.begin_create_or_update(schedule)
```

### Event-Based Triggers
```python
# Trigger on data change
from azure.ai.ml.entities import CronTrigger

trigger = CronTrigger(
    expression="0 2 * * 1"  # Every Monday at 2 AM
)
```

---

## 🔍 Monitoring and Troubleshooting

### Monitoring Pipeline Runs

#### Using Azure ML Studio
1. Navigate to **Jobs** section
2. Click on your pipeline run
3. View step-by-step execution
4. Check logs and outputs for each step

#### Using Python SDK
```python
# Get run details
run_details = ml_client.jobs.get(name=pipeline_run.name)
print(f"Status: {run_details.status}")
print(f"Duration: {run_details.creation_context}")

# List child runs (individual steps)
child_runs = ml_client.jobs.list(parent_job_name=pipeline_run.name)
for child_run in child_runs:
    print(f"Step: {child_run.display_name}, Status: {child_run.status}")
```

### Common Issues and Solutions

| Issue                            | Possible Cause             | Solution                                          |
| -------------------------------- | -------------------------- | ------------------------------------------------- |
| Step fails with "File not found" | Incorrect data path        | Check input paths and datastore connections       |
| Environment errors               | Missing dependencies       | Update environment with required packages         |
| Memory errors                    | Insufficient compute       | Increase compute size or optimize data processing |
| Permission errors                | Insufficient access rights | Check datastore and compute permissions           |
| Timeout errors                   | Long-running operations    | Increase timeout settings or optimize code        |

### Debugging Tips
```python
# Enable detailed logging
import logging
logging.basicConfig(level=logging.DEBUG)

# Add debug outputs to components
@command(
    outputs=dict(debug_info=Output(type="uri_file"))
)
def debug_component():
    # Save debug information
    with open("debug_info.txt", "w") as f:
        f.write("Debug information here")
```

---

## 🎯 Exam Tips

### Key Points to Remember
- [ ] Pipelines enable **automation** and **reproducibility**
- [ ] Components are **reusable** building blocks
- [ ] Data flows between steps using **inputs/outputs**
- [ ] Pipelines can be **scheduled** or **triggered**
- [ ] **Monitoring** is essential for production pipelines

### Common Exam Questions
1. **"How do you pass data between pipeline steps?"**
   - Answer: Using outputs from one step as inputs to another step

2. **"What are the benefits of using components?"**
   - Answer: Reusability, maintainability, version control, testing

3. **"How do you schedule a pipeline to run weekly?"**
   - Answer: Use RecurrenceTrigger with frequency="week"

4. **"What should you do if a pipeline step fails?"**
   - Answer: Check logs, verify inputs, check compute resources, validate environment

---

## 🔗 Related Topics

- [[Model-Training-Scripts]] - Individual training scripts
- [[Model-Management]] - Managing trained models
- [[Azure-ML-Workspace]] - Workspace setup and configuration
- [[Model-Deployment]] - Deploying trained models

---

## 📚 Additional Resources

### Microsoft Documentation
- [Azure ML Pipelines Overview](https://docs.microsoft.com/azure/machine-learning/concept-ml-pipelines)
- [Create ML Pipelines](https://docs.microsoft.com/azure/machine-learning/how-to-create-machine-learning-pipelines)

### Hands-on Labs
- [ ] **Lab 1**: Create a simple 2-step pipeline
- [ ] **Lab 2**: Build custom components
- [ ] **Lab 3**: Schedule pipeline execution
- [ ] **Lab 4**: Debug failed pipeline runs

### Practice Questions
- [ ] How do you create a custom component?
- [ ] What are the different ways to trigger a pipeline?
- [ ] How do you monitor pipeline performance?
- [ ] What data types can be passed between steps?

---

## ✅ Study Checklist

- [ ] Understand pipeline concepts and benefits
- [ ] Know how to create pipelines using Studio, SDK, and YAML
- [ ] Practice passing data between pipeline steps
- [ ] Learn to create custom components
- [ ] Understand scheduling and triggering options
- [ ] Practice monitoring and troubleshooting
- [ ] Complete hands-on labs
- [ ] Review exam-style questions

---

**Tags:** #azure-ml #pipelines #automation #components #scheduling #monitoring
**Exam Objective:** Train and deploy models (25–30%)