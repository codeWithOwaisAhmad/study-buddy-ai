# 🎓 Study Buddy AI — Complete Generative AI & GitOps Guide

> **An intelligent Quiz Generation platform powered by LangChain and Groq's Llama models, featuring a user-friendly Streamlit frontend, containerized with Docker, and deployed via a modern GitOps CI/CD pipeline leveraging Jenkins and ArgoCD onto Kubernetes.**

This isn't just a standard README—it's a **complete end-to-end tutorial**. Whether you want to understand how the LLM dynamically generates structured quizzes or how the DevOps pipeline automatically deploys your code to a Kubernetes cluster on every commit, this guide covers it all.

---

## 📑 Table of Contents

| #  | Section |
|----|---------|
| 1  | [What This Project Actually Does](#-1-what-this-project-actually-does) |
| 2  | [Architecture & Tech Stack](#-2-architecture--tech-stack) |
| 3  | [File-by-File Breakdown](#-3-file-by-file-breakdown) |
| 4  | [Deep Dive: The Application Logic](#-4-deep-dive-the-application-logic) |
| 5  | [The CI/CD Jenkins Pipeline](#-5-the-cicd-jenkins-pipeline--gitops-with-argocd) |
| 6  | [Kubernetes Deployment](#-6-kubernetes-deployment) |
| 7  | [Local Setup & Running the Project](#-7-local-setup--running-the-project) |

---

## 🎯 1. What This Project Actually Does

**Study Buddy AI** is a learning application that helps users study any topic by dynamically generating randomized quizzes. 

When a user opens the app, they can:
1. **Select a input format:** Multiple Choice or Fill-in-the-Blank.
2. **Provide a topic:** e.g., "World War II", "Python Basics", "Geography".
3. **Choose the difficulty & length:** Easy, Medium, Hard, and set 1 to 10 questions.
4. **Take the interactive Quiz:** Answer the questions generated in real-time by the AI.
5. **Get Evaluated:** After submitting, see a detailed breakdown of correct/incorrect answers, the total score percentage, and download the results as a CSV file.

On the infrastructure side, this project implements a true **GitOps Continuous Deployment Pipeline**. Code pushes trigger Jenkins, which builds the Docker image, updates the Kubernetes manifest with the new version tag, and pushes it back to GitHub. ArgoCD detects these changes and automatically updates the live Kubernetes cluster.

---

## 🏗️ 2. Architecture & Tech Stack

| Technology | Role | Why This Choice |
|---|---|---|
| **Python 3.10** | Core Programming Language | Industry standard for AI and data-driven applications. |
| **Streamlit** | Frontend Web UI | Quick, interactive UI generation while maintaining state. |
| **LangChain** | LLM Orchestration | Simplifies prompt engineering, API calls, and parsing. |
| **LangChain-Groq** | LLM Provider Integration | Uses Groq's ultra-fast LPUs for instantaneous API responses. |
| **Pydantic** | Output Parser & Schema Validation | Ensures the LLM strictly responds in the exact JSON structure we need (e.g., questions with exactly 4 options). |
| **Pandas** | Data Management | Used to organize quiz results and export them to a clean CSV. |
| **Docker** | Containerization | Packages the app into an immutable container artifact. |
| **Jenkins** | Continuous Integration | Orchestrates the build, test, and release pipeline. |
| **ArgoCD** | Continuous Deployment (GitOps) | Continuously monitors the Git repo and synchronizes the Kubernetes cluster with the defined `yaml` manifests. |
| **Kubernetes (Minikube)** | App Hosting / Orchestration | Manages the deployments, pods, scaling, and networking. |

---

## 📂 3. File-by-File Breakdown

```
study-buddy-ai/
│
├── Dockerfile                    # Container instructions for the Streamlit app
├── Jenkinsfile                   # Multi-stage automated CI/CD pipeline rules
├── requirements.txt              # Python library dependencies (Langchain, Streamlit, etc.)
├── application.py                # Main Streamlit UI and app initialization
│
├── manifests/                    # Kubernetes definitions for ArgoCD to sync
│   ├── deployment.yaml           # Defines the Pod specs, container image, and secrets
│   └── service.yaml              # Exposes the Deployment using a NodePort service
│
└── src/                          # Core application logic
    ├── generator/
    │   └── question_generator.py # Interfaces with Groq via LangChain, enforces Pydantic schemas
    │
    ├── models/
    │   └── question_schemas.py   # Defines the expected output structures (MCQ, Fill in the Blank)
    │
    ├── utils/
    │   └── helpers.py            # QuizManager class defining state changes and score evaluation
    │
    └── common/
        ├── logger.py             # Reusable custom logging configuration
        └── custom_exception.py   # Specialized error handling 
```

---

## 🔬 4. Deep Dive: The Application Logic

Let's explore the flow of the application in detail.

### 4.1 How the Streamlit UI Manages State (`application.py`)
Streamlit re-runs the entire script from top to bottom every time a button is clicked. To prevent the app from "forgetting" the generated quiz when interacting with it, we heavily rely on `st.session_state`.
- `st.session_state.quiz_manager`: Holds an instance of the `QuizManager` object (defined in helpers), keeping track of the questions and the user's selected answers across re-runs.
- `st.session_state.quiz_generated`: A flag that dictates whether to show the "Generate Quiz" screen or the actual "Quiz Form".

### 4.2 Handling Complex AI Generations (`src/generator/question_generator.py`)
LLMs often struggle with strict formats. If we want an MCQ, we need exactly a question, exactly 4 options, and one correct answer. This project solves this using **PydanticOutputParser**.

```python
def generate_mcq(self, topic: str, difficulty: str) -> MCQQuestion:
    parser = PydanticOutputParser(pydantic_object=MCQQuestion)
    question = self._retry_and_parse(mcq_prompt_template, parser, topic, difficulty)
    
    # Validation step:
    if len(question.options) != 4 or question.correct_answer not in question.options:
        raise ValueError("Invalid MCQ Structure")
    return question
```
- The generator creates a loop (`_retry_and_parse`) that tries to prompt the LLM. 
- The `parser.parse(response.content)` strictly coerces the text output into a Python Object (e.g., `MCQQuestion`).
- If the LLM hallucinates an invalid format, it gets caught and retried, ensuring a bulletproof user experience.

### 4.3 Quiz Evaluation and CSV Download (`src/utils/helpers.py`)
Once the user clicks "Submit", the `QuizManager.evaluate_quiz()` function is triggered.
It compares the user's answers against the `correct_answer` field populated by the LLM. It packs these into a Python List of Dictionaries, converts it to a Pandas DataFrame, scores it as a percentage, and writes it directly to disk via `to_csv()`. The `st.download_button` in `application.py` then links directly to this generated file.

---

## ⚙️ 5. The CI/CD Jenkins Pipeline & GitOps with ArgoCD

The `Jenkinsfile` orchestrates a sophisticated **GitOps workflow**. When a developer commits code, the following happens automatically:

1. **Checkout & Build Phase**: Jenkins pulls the latest code from GitHub and builds a new Docker Image using the standard `Jenkins $BUILD_NUMBER` to uniquely tag it (e.g., `v15`).
2. **DockerHub Push Phase**: It authenticates to DockerHub using configured credentials `dockerhub-token` and pushes the newly minted image.
3. **YAML Update Phase (The GitOps Core)**: Jenkins uses the `sed` command to dynamically overwrite the image tag inside the Kubernetes `manifests/deployment.yaml` file to match the new `$BUILD_NUMBER`.
4. **Git Commit Phase**: Jenkins commits this changed `yaml` file and pushes it back into the GitHub repository's main branch.
5. **ArgoCD Sync Phase**: Jenkins downloads `kubectl` and the `argocd` CLI. It authenticates to an ArgoCD server running in the cluster. It then forcibly tells ArgoCD to sync the `study` app, pulling down the updated `yaml` file it just pushed to GitHub, effectively rolling out the new container version onto the cluster.

---

## ☸️ 6. Kubernetes Deployment

The Kubernetes manifests tell the cluster exactly how to run the application.

### `manifests/deployment.yaml`
- Creates a deployment named `llmops-app` with 2 Pod Replicas for high availability.
- It pulls the image `dataguru97/studybuddy:<TAG>` (This tag is auto-updated by Jenkins).
- It injects environment variables securely via a `SecretKeyRef`. The `GROQ_API_KEY` is completely hidden and loaded from a secret explicitly created in the cluster (`groq-api-secret`).

### `manifests/service.yaml`
- Creates a `NodePort` service named `llmops-service`.
- This routes external HTTP traffic coming to the server port 80 directly to port `8501` on the pods, which is the default Streamlit hosting port exposed in the Dockerfile.

---

## 🚀 7. Local Setup & Running the Project

If you want to run this application locally without Kubernetes or Docker, follow these steps:

### Prerequisites
- Python 3.10+
- A [Groq API Key](https://console.groq.com/keys)

### Step-by-Step Installation
1. **Clone the repository:**
   ```bash
   git clone https://github.com/codeWithOwaisAhmad/study-buddy-ai.git
   cd study-buddy-ai
   ```

2. **Set up virtual environment & install dependencies:**
   ```bash
   python -m venv venv
   source venv/bin/activate    # On Windows: venv\Scripts\activate
   pip install -e .            # Assures src/ modules import correctly via setup.py
   ```

3. **Configure the Environment:**
   Create a `.env` file in the root directory:
   ```env
   GROQ_API_KEY=gsk_your_actual_groq_api_key_here
   ```

4. **Launch the Application:**
   ```bash
   streamlit run application.py
   ```
   Open your browser and navigate to `http://localhost:8501`.

<div align="center">
  <b>Built for modern LLMOps. Containerized, Orchestrated, and Fully GitOps enabled.</b>
</div>